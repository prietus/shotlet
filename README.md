# shotlet

A screenshot and screen recording toolbar for river + rivermonad, modelled
on the one of macOS (⌘⇧5): a floating bar with

- capture the **entire screen**, a **window** or a **selected portion**,
- record the **entire screen** or a **selected portion**,
- **Options**: where to save (Screenshots folder, Desktop, clipboard), a
  5 or 10 second **timer**, mouse pointer, audio, remember the last
  selection, floating thumbnail,
- a **thumbnail** of the result in the corner: click it to annotate the
  screenshot in swappy (or play the recording), right click to dismiss.

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
