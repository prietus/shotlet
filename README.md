# shotlet

A screenshot and screen recording toolbar for river + rivermonad, modelled
on the one of macOS (⌘⇧5): a floating bar with

- capture the **entire screen**, a **window** or a **selected portion**,
- record the **entire screen** or a **selected portion**,
- **Options**: where to save (Screenshots folder, Desktop, clipboard), a
  5 or 10 second **timer**, mouse pointer, audio, remember the last
  selection, floating thumbnail,
- **upload** every screenshot to a pastebin, an image host or your own
  server, and get its address in the clipboard,
- a **thumbnail** of the result in the corner: click it to annotate the
  screenshot in swappy (or play the recording), drag it to another program
  to drop the file there (a terminal gets its path), right click to dismiss.

It is one Python file (GTK 4 on a layer-shell surface); the work is done by
`grim` and `wf-recorder`.

## Use

    shotlet            open the toolbar (again: close it; while recording: stop)
    shotlet MODE       open in screen | window | region | rec-screen | rec-region
    shotlet stop       stop the recording
    shotlet status     "recording SECONDS" or "idle"

| | |
|---|---|
| `1`–`5` | choose the mode |
| `Enter` / `Space` | capture (the screen under the pointer, the window under it or the focused one, the selection) |
| click | capture that screen / that window |
| drag | select a portion; drag it to move, its edges to resize; double click to capture |
| `O` | options |
| `Esc` / right click | close |

Screenshots go to `~/Pictures/Screenshots` (and the clipboard), recordings
to `~/Videos/Recordings`. Settings live in `~/.config/shotlet/config.json`;
`~/.config/shotlet/style.css` (GTK CSS, classes `shotlet-bar`,
`shotlet-menu`, `shotlet-pill`, `shotlet-thumb`) restyles it.

## Uploading

`~/.config/shotlet/uploaders.conf` (written with examples the first time
the toolbar opens) lists the places a screenshot can be uploaded to. Each
appears under **Upload Screenshots to** in Options: tick one and every
screenshot is uploaded after it is taken (click it again to stop). The
address is copied to the clipboard and shown in a notification; a failure
is notified too. Recordings are never uploaded.

```ini
[litterbox (72 h)]
command = curl -fsS -F reqtype=fileupload -F time=72h -F fileToUpload=@{file} https://litterbox.catbox.moe/resources/internals/api.php

[my server]
command = scp -q {file} me@example.org:/srv/http/shots/
url = https://example.org/shots/{name}
```

`command` is run by the shell with `{file}` replaced by the quoted path of
the image; the first `http(s)://` address it prints is the result. If the
answer is JSON, `json` names the key that holds the address; if nothing is
printed, `url` gives it (`{name}` is the file name). A pastebin of your own
with a token:

```ini
[my pastebin]
command = curl -fsS -H "Authorization: Bearer $(cat ~/.config/pastetoken)" -F file=@{file} https://paste.example.org/api/upload
json = url
```

## Install

    pacman -S grim wf-recorder wl-clipboard gtk4 gtk4-layer-shell python-gobject
    pacman -S swappy ffmpeg        # optional: annotate, thumbnails of recordings
    git clone https://github.com/prietus/shotlet ~/shotlet
    ln -s ~/shotlet/shotlet ~/.local/bin/shotlet

- **[rivermonad](https://github.com/prietus/rivermonad)**: bind a key,
  `("M-S-s", spawn "shotlet")`. Window mode asks rivermonad where the
  windows are with the IPC request `"Windows"`; on other compositors the
  remaining modes still work.
- **[waymobar](https://github.com/prietus/waymobar)**: a button that opens
  the toolbar; while a recording runs it shows the duration in red and a
  click stops it.

```haskell
import Control.Exception (SomeException, try)
import Data.IORef (newIORef, readIORef, writeIORef)
import Data.Maybe (fromMaybe)
import GHC.Clock (getMonotonicTime)
import System.Environment (lookupEnv)

screenshot :: Widget
screenshot = spawnOnClick "shotlet" $ periodicIO 1 $ do
  started <- newIORef Nothing              -- when this recording was first seen
  pure $ do
    dir <- fromMaybe "/tmp" <$> lookupEnv "XDG_RUNTIME_DIR"
    pid <- either (const "") (takeWhile (/= '\n')) <$> readFile' (dir ++ "/screenrec")
    alive <- if null pid then pure False
             else either (const False) (const True) <$> readFile' ("/proc/" ++ pid ++ "/comm")
    now <- getMonotonicTime
    if not alive then writeIORef started Nothing >> pure "shot" else do
      t0 <- maybe (writeIORef started (Just now) >> pure now) pure =<< readIORef started
      let secs = round (now - t0) :: Int
          two n = (if n < 10 then "0" else "") ++ show n
      pure ("<span foreground='#f38ba8'>rec " ++ show (secs `div` 60) ++ ":" ++ two (secs `mod` 60) ++ "</span>")
  where
    readFile' :: FilePath -> IO (Either SomeException String)
    readFile' p = try (readFile p >>= \c -> length c `seq` pure c)
```

Any other bar can do the same with `shotlet status` and a click that runs
`shotlet`.

The recording state is `$XDG_RUNTIME_DIR/screenrec` (`PID\nFILE`), the same
file as the older `screenrec` script, so either can stop the other.

## License

BSD 3-Clause, see [LICENSE](LICENSE).
