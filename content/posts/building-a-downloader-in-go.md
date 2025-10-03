---
title           : "Building a Downloader in Go"
description     : >
                  Building a HTTP(s) downloader using the Go standard libray
date            : 2025-03-10
template        : post.html
---

People say Go is well suited for Network applications. With its light Goroutines
and huge standard library it can get most of the job done without even needing a
third-party library.

Great, let me build a terminal program to download files

## Accio
![harry-potter-casting-spell-accio-dittany](https://y.yarn.co/d3a19459-db07-4928-9cbf-dd6111c2722a_text.gif)

The [summoning charm from Harry Potter](https://harrypotter.fandom.com/wiki/Summoning_Charm)
seems like a fit name for my downloader. Like when Harry Potter says
"Accio Dittany" and out pops the bottle of Dittany potion from 
Hermione's bag with an "Undectable Extension Charm", our version of 
Accio would do something similar, albeit with URLs.

So running `accio url` would get you the file from the `url`.

Simple enough! Let's start then.

## The process
The module `net/http` gives us this amazing method for making a GET request -

`http.Get(url string) (resp *httpResponse, err error)`

With this, we can run a `GET` request to `url` and we get a response object (of
type `httpResponse`) and a possible error (of type `error`). We check for any
error that might have occured while making the request and then move on to
reading the data bytes from `httpResponse.Body`.

But before reading the body, we would need to create and open a file for
writing with the following method. And `os` module has the capability we need

`os.OpenFile(name string, flag int, perm fs.FileMode) (*os.File, error)`

we also need `os` to get the CLI argument for URL being passed into our program
by the user while running command `accio URL`. We need `io` module because it
has defined the `io.EOF` error value that we need to test to know when we've
reached the end of reading the response body. With the file opened and the
response body ready for reading, we would simply need to loop this following

### Copy Data process
1. Initialize a buffer
2. Read `n` bytes from response body into a buffer
3. If `n` is non-zero we write `n` bytes to the file
4. If error occured is `EOF` (End of file - indicates response data end)
    
    We break the loop (go to step 6)
5. Go to step 2 (repeat)
6. We close the file and the response body


Here's the actual code

```go
response, err := http.Get(downloadUrl)
// handle error
defer response.Body.Close()

file, err := os.OpenFile(
    "download",
    os.O_WRONLY | os.O_CREATE,
    os.ModePerm,
)
// handle error
defer file.Close()

buffer := make([]byte, 512)
for {
    buffer = buffer[0:512]
    n, err := response.Body.Read(buffer)

    if n > 0 {
        file.Write(buffer[0:n])
    }

    if err != nil {
        if err != io.EOF {
            fmt.Fprintf(
                os.Stderr,
                "download error: %s\n",
                err,
            )
            os.Exit(-1)
        }
        break
    }
}
```

We open the file with `flags` as `os.O_WRONLY | os.O_CREATE` which causes it to
open a file for writing and the `perm` as `os.ModePerm` so that the opened file
gets the same permissions as our executable program `accio`

We reset the `buffer` slice's capacity to `512` each time in the loop to
utilize the full capacity and let the `Read` method of `response.Body` to fill
it up with `n` bytes.

We then proceed to write only `n` bytes to the file because that's how much we 
read in this iteration from `response.Body`

If error has occured, we need to stop everything. If the error is not `EOF` 
we also log this onto the terminal. Otherwise, we silently break the loop

## Showing progress
The next part is to show progress of the download. For this I'm going to use the
channel feature of golang. 

Basically the download (more specifically the copy of bytes from response body
to file) would be happening on another go routine and it will send progress
stats over a channel. The main routine can then read messages from this channel
to display progress.

Here's the structure I used for the download progress status message
```go
type DownloadStatus struct {
	Error           error
	IsComplete      bool
	BytesDownloaded int64
}
```

This will suffice. The copying logic will post a message of type
`DownloadStatus` on the `statusChannel` channel so that the main routine can
display the progress

here's the updated function to copy data from `src` to `dest` while posting
updates of the progress
```go
func copyVerbose(dest io.Writer,
                 src io.Reader,
                 statusChannel chan DownloadStatus) {
	buf := make([]byte, COPY_BUFFER_SIZE)
	status := DownloadStatus{}

	for {
		nread, err := src.Read(buf)
		if nread > 0 {
			nwritten, err := dest.Write(buf[0:nread])
			if err != nil {
				status.Error = err
				break
			}
			status.BytesDownloaded += int64(nwritten)
		}
		if err != nil {
			if err == io.EOF {
				status.IsComplete = true
			} else {
				status.Error = err
			}
		}
		statusChannel <- status
	}
}
```

### Showing progress in 
Now to show progress on the `main` routine, we need to do this

1. create a ticker (which ticks periodically at say 500 ms)
2. use `select` to read from ticker and the `statusChannel`
3. if read from statusChannel, store it in a local variable
4. if read from ticker channel, show the progress
5. go to step 2 and repeat

of course, we would close this repeat loop once we find the status message
contains `IsComplete` as `true`

here's the code for that (partial `main` function)
```go
lastStatus := DownloadStatus{}
ticker:= time.NewTicker(time.Millisecond * 500)
go downloadUrl(url, &DownloadOptions{Filepath: filename}, statusChannel)

done := false
for done != true {
    select {
    case <-ticker.C:
        n, unit := getFormattedSize(lastStatus.BytesDownloaded)
        if lastStatus.IsComplete == true {
            fmt.Printf("\x1B[1K\rcompleted: %.2f %s\n", n, unit)
            done = true
            break
        }
        if lastStatus.Error != nil {
            fmt.Fprintf(
                os.Stderr,
                "download failed: %s\n",
                lastStatus.Error)
            done = true
            break
        }
        fmt.Printf("\x1B[1K\r%.2f %s", n, unit)
    case status := <-statusChannel:
        lastStatus = status
    }
}
```

The escape sequence in printf (`\x1B[1K`) deletes the current line and `\r`
moves the cursor to the beginning of the line. [See
more](https://www2.ccs.neu.edu/research/gpc/VonaUtils/vona/terminal/vtansi.htm)

This effectively shows us the updated progress by deleting the old progress
every 500 ms

## Conclusion
We've got ourselves a http(s) downloader using Golang's standard libraries only.
The downloader shows progress as it downloads and saves the file with name as
found looking at the url (or `download` if the url doesn't contain a
`/resource_name.extension` part at the end

There are many more things that can be done here. Some of the basic ones are
1. Add multi connections to speed up downloads
2. Add pause / resume capability

Both of them requires the usage of response header `Accept-Range` which denotes
whether the server supports requesting file data by a specific range. The
corresponding request header is `Range` which can be added in the request to let
the server know which range of data we need

I'll pick this up in the next iteration of `Accio` our very own http downloader.

## Full code
[https://github.com/riturajborpujari/accio](https://github.com/riturajborpujari/accio)

## References
1. [https://pkg.go.dev/std](https://pkg.go.dev/std)
2. [https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview)
3. [https://www2.ccs.neu.edu/research/gpc/VonaUtils/vona/terminal/vtansi.htm](https://www2.ccs.neu.edu/research/gpc/VonaUtils/vona/terminal/vtansi.htm)
