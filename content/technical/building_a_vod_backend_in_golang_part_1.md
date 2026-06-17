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

### provider

when i say `provider` i mean the underlying storage where we store our uploaded videos to the backend.  we will look at 2 providers in this series

1. Local 
2. Minio

### delivery

when i say `delivery` i mean the underlying delivery where we enable our client to get access to the videos uploaded.  we will look at 2 deliveries in this series

1. Local 
2. Minio

#### interaface details

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

#### video service

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
