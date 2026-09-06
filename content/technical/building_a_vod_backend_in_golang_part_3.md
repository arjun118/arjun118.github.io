+++
date = '2026-08-16T00:37:30+05:30'
draft = true
title = 'Building a Vod Backend in Golang Part 3'
tags=['technical']
+++

# What will be implement

1. postgres db
2. a migration container
3. backend-api
4. backend-worker
5. transactional outbox to solve dual-writes
6. redis stream (we will move away from redis list for our transcode jobs queue)
