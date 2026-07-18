# Fix: Limiting Large Requests in `stdiscosrv`

## What was wrong

The `stdiscosrv` service accepts information from other devices using a POST request.

Before this fix, the service accepted a request of any size. It tried to read the whole request and turn it into a JSON object.

An attacker could send a very large request. The service would keep using more and more memory while reading it. If enough large requests were sent, the service could become slow, stop working, or be killed by the operating system.

The request only needs to contain a small list of device addresses, so there was no good reason to accept an unlimited amount of data.

## How I found it

This was one of the problems found during a security scan of the Syncthing codebase.

The scan checked the places where the program reads data sent over the network. In `cmd/stdiscosrv/apisrv.go`, the POST request was read directly:

```go
json.NewDecoder(req.Body).Decode(&ann)
```

There was no limit on `req.Body` before it was read. That showed that a remote user could send more data than the service needed.

## How I thought about the fix

The fix needed to do three simple things:

1. Stop reading the request after a reasonable size.
2. Keep the current JSON handling.
3. Avoid adding a new helper, package, or custom system.

Go already provides `http.MaxBytesReader` for exactly this purpose. It stops a request body from growing past a chosen size. Using it meant the fix could be kept very small and would work with the existing code.

I chose a limit of 1 MiB. Normal device announcements are much smaller than this, while the limit prevents a request from being needlessly huge.

## What I changed

In `cmd/stdiscosrv/main.go`, I added the limit:

```go
maxAnnouncementSize = 1 << 20
```

This means 1 MiB.

In `cmd/stdiscosrv/apisrv.go`, I placed the limit around the request body before reading the JSON:

```go
json.NewDecoder(http.MaxBytesReader(w, req.Body, maxAnnouncementSize)).Decode(&ann)
```

If a request is too large, reading it fails and the service returns its existing `400 Bad Request` response. The request is rejected before the service can use an unlimited amount of memory.

## How I checked the fix

I ran:

```text
go test ./cmd/stdiscosrv
```

The tests passed.

I also ran `git diff --check`, and it reported no formatting problems.

## GitHub issue check

I used the GitHub command-line tool to search the Syncthing issue list for this problem. I did not find an existing issue matching the missing request-size limit or the memory problem.

## Branch and commit

Branch:

```text
fix/stdiscosrv-post-body-limit
```

Commit:

```text
473c5359 fix: limit stdiscosrv announcement body
```

No pull request was created.
