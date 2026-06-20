+++
date = '2026-06-16T23:21:46+05:30'
draft = false
title = 'Building a Vod Backend in Golang - Part 1'
tags=['technical','golang']
+++

In this series (hope i continue to build this) we wil learn about building a scalable (eventually) video-on-demand backend in golang.

We will go through a few naive implementations until we reach some architecture which is actually scalable (we won't be using any CDN here, but our final pattern will be resembling that, say you can swap out local imlementation of video delivery with cdn easily)

Code will be available at [github](https://github.com/arjun118/vod)

# Setup

this is gonna be a long section. we will set up the directory structure, define some terms which we will use to simplify the explantion
```
 .
├──  docker-compose.yaml
├──  go.mod
├──  go.sum
├──  internal
│   ├──  handlers
│   │   └──  video.go
│   └──  media
│       ├──  delivery
│       │   ├──  cloudfront.go
│       │   └──  local.go
│       ├──  local
│       │   └──  provider.go
│       ├──  minio
│       │   └──  provider.go
│       ├──  provider.go
│       └──  s3
│           └──  provider.go
├──  main.go
├──  service
│   ├──  ffmpeg.go
│   └──  video.go
├──  storage
└──  vids
    └──  vid.mp4
```
1. `main.go`: routes ,spinning the server,entry point
2. `internal`: our core application logic
3. `internal/handlers`: api handlers for our routes
4. `internal/media`: where we implement our various delivery and provider interface
5. `internal/media/provider.go`: delivery, provider interface will be defined here 
5. `service`: `video.go` has convenience structs which will encapsulate the delivery and provider implementations for us to access common methods across our api handlers irrespective of the provider and delivery implementations we use.`ffmpeg.go` just have ffmpeg functionality
6. `storage`: used to store the uploaded videos when our local is the provider
7. `vids`: you don't really need this, this is the video i will be uploading to test out the functionality.

## Terms

### Provider

when i say `provider` i mean the underlying storage where we store our uploaded videos to the backend.  we will look at 2 providers in this series

1. Local 
2. Minio

### Delivery

when i say `delivery` i mean the underlying delivery where we enable our client to get access to the videos uploaded.  we will look at 2 deliveries in this series

1. Local 
2. Minio

#### Interaface Details

in : `internal/media/provider.go`

```go
package media

import (
	"context"
	"io"
)

type FileMetaData struct {
	Filename    string
	ContentType string
	Size        int64
	OwnerID     string
}

type MediaObject struct {
	ID           string `json:"id"`
	ObjectKey    string `json:"key"`
	Provider     string `json:"provider"`
	ContentType  string `json:"content_type"`
	Size         int64  `json:"size"`
	PlaybackURL  string `json:"playback_url"`
	ThumbnailURL string `json:"thumbnail_url"`
}

type StorageProvider interface {
	Save(ctx context.Context, objectKey string, r io.Reader, meta FileMetaData) (int64, error)
	Delete(ctx context.Context, objectKey string) error
}

type DeliveryProvider interface {
	URL(ctx context.Context, objectKey string) (string, error)
}
```

our local and minio implementations for delivery and provider will satisfy these interfaces, hence abstracting complexity when used with VideoService

#### Video Service

```go
package service

import (
	"context"
	"fmt"
	"io"
	"log"
	"os"
	"path/filepath"
	"sync"
	"time"

	"github.com/arjun118/fileupload/internal/media"
	"github.com/google/uuid"
)

type TranscodeJob struct {
	ID             string
	TempSourcePath string
	FinalBasePath  string
	Ctx            context.Context
}

type VideoService struct {
	storage    media.StorageProvider
	delivery   media.DeliveryProvider
	jobQueue   chan TranscodeJob
	workerWg   sync.WaitGroup
	maxWorkers int
}

func NewVideoService(storage media.StorageProvider, delivery media.DeliveryProvider, maxWorkers int) *VideoService {
	svc := &VideoService{
		storage:  storage,
		delivery: delivery,
		//buffer size for queued jobs
		jobQueue:   make(chan TranscodeJob, 100),
		maxWorkers: maxWorkers,
	}
	svc.StartWorkerPool()
	return svc
}

func (v *VideoService) StartWorkerPool() {
	for i := range v.maxWorkers {
		v.workerWg.Add(1)
		go func(workerID int) {
			defer v.workerWg.Done()
			log.Printf("starting transcode worker %d", workerID)
			for job := range v.jobQueue {
				log.Printf("Worker %d starting job %s", workerID, job.ID)
				jobCtx, cancel := context.WithTimeout(context.Background(), 10*time.Minute)
				err := v.processAndSaveHLS(jobCtx, job.ID, job.TempSourcePath, job.FinalBasePath)
				if err != nil {
					log.Printf("[Error] Job %s failed: %v", job.ID, err)
					//prod: update database job status - failed
				} else {
					log.Printf("Job %s completed successfully", job.ID)
				}
				cancel()
			}
		}(i)
	}
}

func (v *VideoService) StopWorkerPool() {
	close(v.jobQueue)
	v.workerWg.Wait()
}

func (v *VideoService) generateHLSFolderPath(videoID string) string {
	now := time.Now()
	year := now.Format("2006")
	month := now.Format("01")

	return filepath.ToSlash(filepath.Join("videos", year, month, videoID))
}

func (v *VideoService) processAndSaveHLS(ctx context.Context, videoID string, tempSourcePath string, finalBasePath string) (err error) {
	tempHLSDir := filepath.Join("/tmp", "hls_"+videoID)

	if err := os.MkdirAll(tempHLSDir, 0755); err != nil {
		return fmt.Errorf("failed to create HLS temp dir: %w", err)
	}
	//clean the temporary folders
	defer func() {
		os.RemoveAll(tempHLSDir)
		os.Remove(tempSourcePath)
		if r := recover(); r != nil {
			err = fmt.Errorf("panic recovered during transcoding: %v", r)
		}
	}()

	_, err = TranscodeToHLS(ctx, tempSourcePath, tempHLSDir)
	if err != nil {
		return fmt.Errorf("transcode failed: %w", err)
	}
	entries, err := os.ReadDir(tempHLSDir)
	if err != nil {
		return fmt.Errorf("failed to read output directory %w", err)
	}
	for _, entry := range entries {

		if entry.IsDir() {
			continue
		}
		//playlist.m3u8 or playlist0.ts
		fileName := entry.Name()
		tempFilePath := filepath.Join(tempHLSDir, fileName)
		finalObjectKey := filepath.ToSlash(filepath.Join(finalBasePath, fileName))
		if err := v.saveSegment(ctx, tempFilePath, finalObjectKey, fileName); err != nil {
			return fmt.Errorf("failed to save segment %s: %w", fileName, err)
		}
	}
	return nil
}

func (v *VideoService) saveSegment(ctx context.Context, srcPath, destKey, fileName string) error {
	file, err := os.Open(srcPath)
	if err != nil {
		return err
	}
	defer file.Close()
	contentType := "video/mp2t"
	if filepath.Ext(fileName) == ".m3u8" {
		contentType = "application/vnd.apple.mpegurl"
	}
	meta := media.FileMetaData{
		Filename:    fileName,
		ContentType: contentType,
	}
	_, err = v.storage.Save(ctx, destKey, file, meta)
	if err != nil {
		return fmt.Errorf("storage save failed: %w", err)
	}
	return nil
}

func (v *VideoService) Upload(ctx context.Context, file io.Reader, meta media.FileMetaData) (*media.MediaObject, error) {
	//a uuid string
	videoID := uuid.NewString()
	// save the raw file to the disk right now
	tempRawDir := filepath.Join("/tmp", "raw_videos")
	if err := os.MkdirAll(tempRawDir, 0755); err != nil {
		return nil, err
	}
	tempSourcePath := filepath.Join(tempRawDir, videoID+".mp4")
	tempFile, err := os.Create(tempSourcePath)
	if err != nil {
		return nil, err
	}
	log.Println("writing the raw file to temp file: ", tempSourcePath)
	size, err := io.Copy(tempFile, file)
	log.Println("size of the raw file: ", float64(size)/float64(1024*1024))
	tempFile.Close()
	if err != nil {
		os.Remove(tempSourcePath)
		return nil, fmt.Errorf("failed to save raw file upload: %w", err)
	}
	// videos/2026/06/video_uuid
	finalBasePath := v.generateHLSFolderPath(videoID)
	// videos/2026/06/video_uuid/playlist.m3u8
	playlistKey := filepath.ToSlash(filepath.Join(finalBasePath, "playlist.m3u8"))
	playBackURL, err := v.delivery.URL(ctx, playlistKey)
	if err != nil {
		os.Remove(tempSourcePath)
		return nil, fmt.Errorf("failed to generate playback URL: %w", err)
	}
	job := TranscodeJob{
		ID:             videoID,
		TempSourcePath: tempSourcePath,
		FinalBasePath:  finalBasePath,
		Ctx:            context.Background(),
	}
	select {
	case v.jobQueue <- job:
		log.Printf("Job %s successfully queued: ", videoID)
	default:
		//queue is full, handle backpressure
		os.Remove(tempSourcePath)
		return nil, fmt.Errorf("server queue is full, please retry again")
	}
	return &media.MediaObject{
		ID:          videoID,
		ObjectKey:   playlistKey,
		Provider:    "local",
		ContentType: "application/vnd.apple.mpegurl",
		Size:        size,
		PlaybackURL: playBackURL,
	}, nil
}

func (s *VideoService) GetPlaybackURL(ctx context.Context, objectKey string) (string, error) {
	// Business logic goes here:
	// e.g., s.db.GetVideoOwner(objectKey) -> check if current user is allowed to view it.

	url, err := s.delivery.URL(ctx, objectKey)
	if err != nil {
		return "", fmt.Errorf("failed to get playback URL for key %s: %w", objectKey, err)
	}

	return url, nil
}
func (s *VideoService) Delete(ctx context.Context, objectKey string) error {
	// Business logic goes here:
	// e.g., check if the user requesting the delete actually owns the video.
	// 1. Delete from physical storage
	err := s.storage.Delete(ctx, objectKey)
	if err != nil {
		return fmt.Errorf("failed to delete video from storage: %w", err)
	}
	// 2. Future: Delete metadata from database
	// err = s.db.DeleteVideo(ctx, objectKey)
	// if err != nil { ... }
	return nil
}
```
we will do less of a code explanation but more of `how the request flows from end to end`


# Streaming and Transcoding

I will eventually confuse you if i try to explain this.Please, I beg you to read this series of articles to understand video streaming, transcoding to hls segments

[this helped me a lot in understanding streaming and hls in general](https://medium.com/@o.rasouli92/building-a-vod-platform-with-go-and-ffmpeg-part-1-foundations-771e1e14f79b)

Please skim-through that series before continuing further

# Implementations

## Local Storage and Delivery (`branch: main`)

1. we will save the files to uploaded to our backend will be stored to our local file system and when client requests we will deliver from our local system (network i/o of the video segments will be done our backend)

Lets look at the local storage provider implementation

### Provider

in: `internal/media/local/provider.go`

```go
package local

import (
	"context"
	"io"
	"os"
	"path/filepath"

	"github.com/arjun118/fileupload/internal/media"
)

type Storage struct {
	RootDir string
}

func NewStorage(rootDIR string) *Storage {
	return &Storage{
		RootDir: rootDIR,
	}
}

func (p *Storage) Save(ctx context.Context, objectKey string, r io.Reader, meta media.FileMetaData) (int64, error) {
	fullPath := filepath.Join(p.RootDir, filepath.FromSlash(objectKey))

	if err := os.MkdirAll(filepath.Dir(fullPath), 0o755); err != nil {
		return 0, err
	}
	f, err := os.Create(fullPath)
	if err != nil {
		return 0, err
	}
	defer f.Close()

	size, err := io.Copy(f, r)
	if err != nil {
		return 0, err
	}
	return size, nil
}

func (p *Storage) Delete(ctx context.Context, objectKey string) error {
	fullPath := filepath.Join(p.RootDir, filepath.FromSlash(objectKey))
	if err := os.Remove(fullPath); err != nil && !os.IsNotExist(err) {
		return err
	}
	return nil
}
```

### Delivery

and the delivery implementation

in: `internal/media/delivery/local.go`

```go
package delivery

import (
	"context"
	"fmt"
	"strings"
)

type LocalDelivery struct {
	BaseURL string
}

func NewLocalDelivery(baseURL string) *LocalDelivery {
	return &LocalDelivery{
		BaseURL: baseURL,
	}
}

func (d *LocalDelivery) URL(ctx context.Context, objectKey string) (string, error) {
	return d.playbackURL(objectKey), nil
}

func (d *LocalDelivery) playbackURL(objectKey string) string {
	return fmt.Sprintf("%s/%s", strings.TrimRight(d.BaseURL, "/"), objectKey)
}
```

### Server

let see how our `main.go` acts here, i am only posting the routes and help you understand how our fileserver helps

server implementation, in : `main.go`

```go
storageDir := "./storage"
storageProvider := local.NewStorage(storageDir)
deliverProvider := delivery.NewLocalDelivery("http://localhost:8080/media")
videoService := service.NewVideoService(storageProvider, deliverProvider, 3)
videoHandler := handlers.NewVideoHandler(videoService)
r := chi.NewRouter()
r.Use(middleware.Logger)
r.Use(middleware.Recoverer)
r.Route("/api/videos", func(r chi.Router) {
	r.Post("/", videoHandler.Upload)
	r.Get("/url", videoHandler.GetURL)
	r.Delete("/", videoHandler.Delete)
})
workDir, _ := os.Getwd()
filesDir := http.Dir(filepath.Join(workDir, "storage"))
r.Handle("/media/*", http.StripPrefix("/media", http.FileServer(filesDir)))

server := &http.Server{
	Addr:    ":8080",
	Handler: r,
}
```

In local development, the `LocalStorageProvider` persists uploaded media to the `storage/` directory on disk. We then mount that directory using `http.FileServer`, making it our delivery mechanism. This allows the generated playback URLs to point to `/media/...`, while the actual files are served directly from the filesystem without requiring custom streaming logic.

### Request Lifecycle

![local storage and local delivery arch](/images/local_storage_local_delivery.png)

1. when client (browser or any frontend) requests for this particular file - it will do as below
2. client request: `http://localhost:8080/media/videos/2026/06/abc123/playlist.m3u8`
3. after StripPrefix: `/videos/2026/06/abc123/playlist.m3u8` (ignorning the http://....)
4. resolved on disk: `./storage/videos/2026/06/abc123/playlist.m3u8` - where our actual hls related files are stored

### Tradeoffs

This implementation is very naive and simple and has several limitations.

#### 1. Application Server Handles Everything
Our Go application is responsible for:
- Accepting uploads
- Writing files to disk
- Running FFmpeg transcoding jobs
- Serving HLS playlists
- Serving HLS segments

This means both `CPU-intensive work (transcoding)` and `network-intensive work (video delivery)` happen on the same machine.

As concurrent viewers increase, the application server starts spending more time moving video bytes than executing business logic.

#### 2. Video Traffic Flows Through The Backend
Every HLS request:
```text
GET playlist.m3u8
GET segment000.ts
GET segment001.ts
GET segment002.ts
...
```
hits our Go server.
For a single viewer this is fine. For hundreds or thousands of viewers, our backend becomes a media relay rather than an API server.

#### 3. Application Restarts Become Risky

The local filesystem becomes the source of truth.
If:
- the server is replaced
- the disk is lost
- a container is recreated without persistent volumes
uploaded videos disappear.
Object storage systems such as MinIO or S3 are designed specifically to solve this problem.

i hope that the request lifecycle is clear. it wil almost remain the same in our upcoming implementations but the responsibilities of video-delivery will shift away from our golang application server.

### Working

![local storage and delivery demo](/images/local_storage_local_delivery_demo.png)

1. start the server using `go run main.go`
2. send a request : `curl -X POST http://localhost:8080/api/videos -F "file=@/path/to/your-video"`
3. open the playback url in the browser of your choice
4. you can see the browser handles fetching the master file and the subsequent segments for streaming

## Public Minio storage and delivery (`branch: public_minio_hls`)

In this implementation we will introduce [Minio](https://www.min.io/)

our storage and delivery will be handeld by Minio leaving our golang application server handle the actual business logic

### Provider

in: `internal/media/minio/provider.go`

```golang
package minio

import (
	"context"
	"fmt"
	"io"
	"log"
	"path/filepath"

	"github.com/arjun118/fileupload/internal/media"
	"github.com/minio/minio-go/v7"
)

// put object
// remove object
// stat object

type Storage struct {
	Client     *minio.Client
	BucketName string
}

func NewStorage(client *minio.Client, bucketName string) *Storage {
	return &Storage{
		Client:     client,
		BucketName: bucketName,
	}
}

func (s *Storage) EnsureBucket(ctx context.Context) error {
	bucketExists, err := s.Client.BucketExists(ctx, s.BucketName)
	if err != nil {
		return fmt.Errorf("failed to check bucket existence: %w", err)
	}
	if !bucketExists {
		log.Printf("bucket doesnot exists, creating bucket: %s\n", s.BucketName)
		err := s.Client.MakeBucket(ctx, s.BucketName, minio.MakeBucketOptions{})
		if err != nil {
			return fmt.Errorf("failed to create bucket: %w", err)
		}
	}
	//you might want to configure cors rules for your bucket while using the backend with a frontend
	publicPolicy := fmt.Sprintf(`{
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Sid": "PublicRead",
                    "Principal": {"AWS": ["*"]},
                    "Action": ["s3:GetObject"],
                    "Resource": ["arn:aws:s3:::%s/*"]
                }
            ]
        }`, s.BucketName)

	log.Println("Configuring public read policy for bucket:", s.BucketName)
	if err = s.Client.SetBucketPolicy(ctx, s.BucketName, publicPolicy); err !=
		nil {
		return fmt.Errorf("failed to set public policy: %w", err)
	}
	return nil
}

func (s *Storage) Save(ctx context.Context, objectKey string, r io.Reader, meta media.FileMetaData) (int64, error) {
	//save this to minio bucket
	// videos/year/month/filename.ext
	filePathWithFolder := filepath.FromSlash(objectKey)

	var contentType string
	switch filepath.Ext(objectKey) {
	case ".m3u8":
		contentType = "application/vnd.apple.mpegurl"
	case ".ts":
		contentType = "video/mp2t"
	case ".vtt":
		contentType = "text/vtt"
	case ".jpg", ".jpeg":
		contentType = "image/jpeg"
	case ".png":
		contentType = "image/png"
	default:
		contentType = "application/octet-stream"
	}

	uploadInfo, err := s.Client.PutObject(ctx, s.BucketName, filePathWithFolder,
		r, -1, minio.PutObjectOptions{
			ContentType: contentType,
		})
	if err != nil {
		return 0, err
	}
	size := uploadInfo.Size

	return size, nil
}

func (s *Storage) Delete(ctx context.Context, objectKey string) error {
	//delete from bucket
	filePathWithFolder := filepath.FromSlash(objectKey)
	bucketExists, err := s.Client.BucketExists(ctx, s.BucketName)
	if err != nil {
		return err
	}
	if bucketExists {
		_, err := s.Client.StatObject(ctx, s.BucketName, filePathWithFolder, minio.StatObjectOptions{})
		if err != nil {
			return err
		}
	}
	err = s.Client.RemoveObject(ctx, s.BucketName, filePathWithFolder, minio.RemoveObjectOptions{})
	return err
}
```

1. our minio porvider is simple, it contains functions to ensure a particular bucket, save objects to that bucket,delete from the bucket
2. `EnsureBucket`: it will create a bucket if it doesnot exist. if it exits it will be left as is. while creating the bucket we will set it to be accessible by anyone who has the playback url. i.e `public`. no access control is implemented.
3. `Save`: we wills save the objects to our minio storage preserving their contenttype so that while minio acts as a provider it doesnot send `ContentType: application/octet-stream` which will break our video (browser cannot make sense of the master manifest(`.m3u8`) and segments(`.ts`) content without the correct `ContentType` header). setting this while saving our objects to minio directly helps us to stream the video correctly from minio

### Delivery

in: `internal/media/delivery/minio.go`

delivery is very simple, we just put together the URL

```golang
package delivery

import (
	"context"
	"fmt"
	"log"
	"strings"
)

type MinioDelivery struct {
	BucketName string
	EndPoint   string
}

func NewMinioDelivery(bucketName string, endpoint string) *MinioDelivery {
	return &MinioDelivery{
		BucketName: bucketName,
		EndPoint:   endpoint,
	}
}

func (m *MinioDelivery) URL(ctx context.Context, objectKey string) (string, error) {
	return m.playbackURL(objectKey), nil
}

func (m *MinioDelivery) playbackURL(objectKey string) string {
	url := fmt.Sprintf("http://%s/%s/%s", strings.TrimRight(m.EndPoint, "/"), m.BucketName, objectKey)
	log.Println("playback url called with objectkey: ", objectKey, "url: ", url)
	return url
}
```

### Server

```golang
bucketName := "storage"
	endpoint := "minio:9000"
	accessKeyID := "adminpass"
	secretAccessKey := "adminpass"
	useSSL := false
	minioClient, err := miniosdk.New(endpoint, &miniosdk.Options{
		Creds:  credentials.NewStaticV4(accessKeyID, secretAccessKey, ""),
		Secure: useSSL,
	})
	if err != nil {
		log.Fatalln(err)
	}
	fmt.Println("connected to minio successfully")
	storageProvider := minio.NewStorage(minioClient, bucketName)
	for i := 0; i < 30; i++ {

		err = storageProvider.EnsureBucket(context.Background())

		if err == nil {
			break
		}

		log.Printf("waiting for minio (%d/30): %v", i+1, err)

		time.Sleep(2 * time.Second)
	}

	if err != nil {
		log.Fatalf("failed to initialize bucket: %v", err)
	} else {
		log.Println("ensured bucket...")
	}
	deliverProvider := delivery.NewMinioDelivery(bucketName, "localhost:9000")
	videoService := service.NewVideoService(storageProvider, deliverProvider, 3, "minio")
	videoHandler := handlers.NewVideoHandler(videoService)
	r := chi.NewRouter()
	r.Use(middleware.Logger)
	r.Use(middleware.Recoverer)
	r.Route("/api/videos", func(r chi.Router) {
		r.Post("/", videoHandler.Upload)   // POST /api/videos
		r.Get("/url", videoHandler.GetURL) // GET /api/videos/url?key=...
		r.Delete("/", videoHandler.Delete) // DELETE /api/videos?key=...
	})
	server := &http.Server{
		Addr:    ":8080",
		Handler: r,
	}
```

1. notice that there is no fileserver in our `main.go`, that is because our application server doesnot handle the video delivery directly. it is totally handled by our minio instance
2. the `api endpoint` and the `playback url` are different, because our minio delivery endpoint and api endpoint don't talk to each other or not routed in anyway

we will introduce docker files from now, to get our backend and minio instance up and running

docker manifest for our application api (backend)

in:  `internal/Dockerfile`

```dockerfile
FROM golang:1.26.1 AS builder

WORKDIR /app

COPY . .

COPY go.mod main.go ./

RUN go mod download

COPY internal ./internal
COPY service ./service

RUN CGO_ENABLED=0 GOOS=linux go build -o server .

FROM alpine:latest

WORKDIR /app

RUN apk add --no-cache ffmpeg

COPY --from=builder /app/server .

CMD ["./server"]
```

in: `docker-compose-yaml`

```dockerfile
services:
  minio:
    image: quay.io/minio/minio
    container_name: minio
    restart: always
    ports:
      - "9000:9000"
      - "9001:9001"
    environment:
      MINIO_ROOT_USER: adminpass
      MINIO_ROOT_PASSWORD: adminpass
    command: server --console-address ":9001" /data
    volumes:
      - ~/minio-data:/data

  backend:
    build:
      context: .
      dockerfile: internal/Dockerfile
    depends_on:
      - minio
    ports:
      - "8080:8080"
```

### Request Lifecycle

![public minio hls storage and delivery](/images/public_minio_hls_storage_and_delivery.png)

### Tradeoffs
1. raw videos are still stored on the application server
2. public-minio bucket will have unrestricted access to the videos - which is not desired in most of the cases
3. our in-memory buffered channel is not durable - the transcoding job is tightly coupled with our application logic (we will focus on this much later in this series). i will not highlight this as a con in the upcoming implementation because this will stay the same for all our implementation until we get to our actual architectural changes, which we will do.
4. transcoding is still coupled with our application logic and our application server, while `network-io` video delivery is completely offloaded to our minio instance.


### Working

![public minio hls demo](/images/public_minio_hls_demo.png)

1.1. start the app using `docker compose up --build`
2. send a request : `curl -X POST http://localhost:8080/api/videos -F "file=@/path/to/your-video"`
3. open the playback url in the browser of your choice
4. notice that the playback url will be different from that of our application url (port is different) because the delivery is entirely handled by minio.eg: `http://localhost:9000/storage/videos/2026/06/0a1ba2c8-7e89-43e2-9edc-1f43d2df6fd2/playlist.m3u8`
5. you can see the browser handles fetching the mainfest file and the subsequent segments for streaming

## Private Minio Storage - Go Delivery (`branch: private_go_streaming`)

### Provider

the provider will remain the exact same as our previous public implementation but the only will be that we won't change our bucket to be public. we leave it as it is - `default: private`

the change is only in the `Save` method

```golang
func (s *Storage) Save(ctx context.Context, objectKey string, r io.Reader, meta media.FileMetaData) (int64, error) {
	//save this to minio bucket
	// videos/year/month/filename.ext
	filePathWithFolder := filepath.FromSlash(objectKey)

	var contentType string
	switch filepath.Ext(objectKey) {
	case ".m3u8":
		contentType = "application/vnd.apple.mpegurl"
	case ".ts":
		contentType = "video/mp2t"
	case ".vtt":
		contentType = "text/vtt"
	case ".jpg", ".jpeg":
		contentType = "image/jpeg"
	case ".png":
		contentType = "image/png"
	default:
		contentType = "application/octet-stream"
	}

	uploadInfo, err := s.Client.PutObject(ctx, s.BucketName, filePathWithFolder,
		r, -1, minio.PutObjectOptions{
			ContentType: contentType,
		})
	if err != nil {
		return 0, err
	}
	size := uploadInfo.Size

	return size, nil
}
```

### Delivery

our minio delivery will also remain the same.

### Server

this is where the major change will be seen.

we will be essentially be routing the requests for playback through our application server.

```golang
bucketName := "storage"
	endpoint := "minio:9000"
	accessKeyID := "adminpass"
	secretAccessKey := "adminpass"
	useSSL := false
	minioClient, err := miniosdk.New(endpoint, &miniosdk.Options{
		Creds:  credentials.NewStaticV4(accessKeyID, secretAccessKey, ""),
		Secure: useSSL,
	})
	if err != nil {
		log.Fatalln(err)
	}

	storageProvider := minio.NewStorage(minioClient, bucketName)
	for i := 0; i < 30; i++ {
		err = storageProvider.EnsureBucket(context.Background())

		if err == nil {
			break
		}

		log.Printf("waiting for minio (%d/30): %v", i+1, err)

		time.Sleep(2 * time.Second)
	}

	if err != nil {
		log.Fatalf("failed to initialize bucket: %v", err)
	} else {
		log.Println("ensured bucket...")
	}
	deliverProvider := delivery.NewMinioDelivery(bucketName, "localhost:8080/media")
	videoService := service.NewVideoService(storageProvider, deliverProvider, 3, "minio")
	videoHandler := handlers.NewVideoHandler(videoService)
	r := chi.NewRouter()
	r.Use(middleware.Logger)
	r.Use(middleware.Recoverer)
	r.Route("/api/videos", func(r chi.Router) {
		r.Post("/", videoHandler.Upload)   // POST /api/videos
		r.Get("/url", videoHandler.GetURL) // GET /api/videos/url?key=...
		r.Delete("/", videoHandler.Delete) // DELETE /api/videos?key=...
	})
	r.Handle("/media/*", &handlers.ProxyHandler{
		MinioClient: minioClient,
		BucketName:  bucketName,
	})
	server := &http.Server{
		Addr:    ":8080",
		Handler: r,
	}
```

1. look at the `ProxyHandler`. this is our handler which handles the media delivery requests.

lets look at the `ServeHTTP` method on our handler

in: `internal/handlers/proxy.go`

```golang
package handlers

import (
	"io"
	"net/http"
	"path/filepath"
	"strconv"
	"strings"

	"github.com/minio/minio-go/v7"
)

//cache control to be implemented

type ProxyHandler struct {
	MinioClient *minio.Client
	BucketName  string
}

func (p *ProxyHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodGet &&
		r.Method != http.MethodHead {
		http.Error(
			w,
			"method not allowed",
			http.StatusMethodNotAllowed,
		)
		return
	}
	objectKey := strings.TrimPrefix(r.URL.Path, "/media/")
	if objectKey == "" {
		http.Error(w, "missing object key", http.StatusBadRequest)
		return
	}

	obj, err := p.MinioClient.GetObject(r.Context(), p.BucketName, objectKey, minio.GetObjectOptions{})
	if err != nil {
		http.Error(w, "file not found", http.StatusNotFound)
		return
	}
	defer obj.Close()
	info, err := obj.Stat()
	if err != nil {
		http.Error(w, "not found", http.StatusNotFound)
		return
	}
	contentType := info.ContentType
	if contentType == "" {
		switch filepath.Ext(objectKey) {
		case ".m3u8":
			contentType = "application/vnd.apple.mpegurl"
		case ".ts":
			contentType = "video/mp2t"
		default:
			contentType = "application/octet-stream"
		}
	}
	//segment not modified
	if r.Header.Get("If-None-Match") == info.ETag {
		w.WriteHeader(http.StatusNotModified)
		return
	}
	w.Header().Set("Content-Type", contentType)
	w.Header().Set("Content-Length", strconv.FormatInt(info.Size, 10))
	w.Header().Set("ETag", info.ETag)
	w.Header().Set("Last-Modified", info.LastModified.UTC().Format(http.TimeFormat))
	if r.Method == http.MethodHead {
		w.WriteHeader(http.StatusOK)
		return
	}
	_, _ = io.Copy(w, obj)
}
```

i will explain how the request the video segments will be sent to the client when requested - through our application server - refer to request lifecycle diagram below

### Request Lifecycle

![private_minio_go_delivery](/images/private_minio_go_delivery.png)

### Tradeoffs

1. while the minio bucket is private now, the go proxy handles our video delivery.
2. request for every video segment goes through our application server - hence , the `network i/o overhead of video delivery` has came back. 

### Working

![private_minio_go_delivery_demo](/images/private_minio_go_delivery_demo.png)

1. observe the playback url is : `http://localhost:8080/media/videos/2026/06/6b40bb6d-1782-4298-b5e3-ee2f95057f48/playlist.m3u8`


## Closing

we will see our nginx implementations in the part-2 of this series and part-3 will contain architectural improvements to make our backend scalable and seperate responsibilities (low coupling)
