# DroidCam View

A small Linux desktop app that uses an Android phone running
[DroidCam](https://www.dev47apps.com/) as a webcam. It connects to the phone
over WiFi, shows a live preview, and lets you rotate, mirror, flip and zoom
the picture, which the official Linux client cannot fully do.

The transformed picture is also sent to a virtual camera, so browsers and
video-call apps can use it like any other webcam.

## Features

- Live preview of the phone's camera
- Rotate left / right in 90° steps
- Mirror (horizontal flip) and vertical flip
- Digital zoom from 1× to 4×, with drag-to-reframe
- Fixed picture shapes: 16:9, 4:3, 1:1, 3:4 and 9:16
- Removes the date stamp the phone draws on its picture
- Output to a v4l2loopback virtual camera
- Remembers the last phone address, orientation, shape, zoom and framing

Video only: the phone's microphone is not used, and USB connections are not
supported.

## Requirements

- Linux with a graphical desktop (developed on Fedora `44 with GNOME)
- Python 3
- GTK 4 and libadwaita, with their Python bindings (PyGObject)`
- GStreamer 1.x with the *base* and *good* plugin sets and the GTK 4 video
  sink (`gtk4paintablesink`)
- The DroidCam app on the phone, with phone and PC on the same network
- Optional: the `v4l2loopback` kernel module. Without it the app still shows
  the preview, but other apps cannot see the camera.

The official DroidCam Linux client is **not** required.

### Installing on Fedora

```
sudo dnf install python3-gobject gtk4 libadwaita \
    gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugin-gtk4
```

For the virtual camera, install v4l2loopback from
[RPM Fusion](https://rpmfusion.org/Configuration) and load it:

```
sudo dnf install akmod-v4l2loopback v4l2loopback
sudo modprobe v4l2loopback exclusive_caps=1 card_label="DroidCam"
```

To load it at every boot with the same options:

```
echo v4l2loopback | sudo tee /etc/modules-load.d/v4l2loopback.conf
echo 'options v4l2loopback exclusive_caps=1 card_label="DroidCam"' \
    | sudo tee /etc/modprobe.d/v4l2loopback.conf
```

`exclusive_caps=1` is needed for Chrome and other browsers to list the camera.

## Usage

1. Open DroidCam on the phone and note the IP address and port it shows
   (the port is normally 4747).
2. Close the official DroidCam client on the PC if it is running.
3. Start the app:

   ```
   ./droidcam-view
   ```

4. Enter the phone's IP and port and press **Connect**.
5. In the browser or video-call app, select the camera named **DroidCam**.

`./droidcam-view --connect` connects immediately to the last phone used.

### Adding it to the app menu

To start it from GNOME's "Show Apps" grid, create
`~/.local/share/applications/local.droidcam.View.desktop`, with `Exec`
pointing to where `droidcam-view` lives:

```
[Desktop Entry]
Type=Application
Name=DroidCam View
Comment=Use your phone as a webcam, with rotate, flip and zoom
Exec=/path/to/droidcam-view
Icon=camera-web
Terminal=false
Categories=AudioVideo;Video;
Keywords=droidcam;webcam;camera;phone;
StartupNotify=true
```

The file name must match the app ID (`local.droidcam.View`) so the desktop
shows the right icon for the open window.

### Controls

| Control | Effect |
| --- | --- |
| Rotate left / right | Turns the picture 90°. The stream reconnects briefly. |
| Mirror | Flips the picture horizontally. |
| Flip vertically | Turns the picture upside down. |
| Zoom − / + | Zooms out or in by 0.5×. |
| Zoom level (e.g. `2×`) | Resets the zoom to 1× and centres the framing. |
| Drag the preview | When zoomed in, moves the picture to adjust the framing. |
| Double-click the preview | Centres the framing. |
| Shape (top right) | Crops the picture to a fixed shape. *Original* keeps the phone's. The stream reconnects briefly. |

Mirror, vertical flip, zoom and framing apply instantly, to the preview and
the virtual camera alike.

## How it works

The DroidCam phone app serves its video as an MJPEG stream at
`http://<phone-ip>:<port>/video`. The app reads that stream with a GStreamer
pipeline, crops and rescales it for zoom, rotates and flips it, then splits
it to the preview window and to the first v4l2loopback device.

Settings are stored in `~/.config/droidcam-view.json`.

## Notes and limitations

- A 90° turn or a new shape changes the picture's size (for example 640x480
  becomes 480x640). An app that is already using the camera may need to
  reselect it or reload the page.
- Shapes apply to the picture as you see it, after rotating. A shape far from
  the phone's own, such as 16:9 from a phone held upright, crops a lot and
  leaves a smaller picture (464x260 from 480x640).
- Zoom is digital: it crops the centre of the picture and scales it up, so
  the picture gets softer at higher levels.
- The picture resolution is whatever the phone app sends, 640x480 by default.
- The phone stamps "DroidCam Video Feed" and the date along the top edge of
  its picture. The app always crops off the top 3% of the picture to hide
  it, which also means a very slight zoom at 1×.
- The official DroidCam client leaves the virtual camera locked to its last
  picture format. The app clears that lock each time it connects.
- A virtual camera named "DroidCam Rotated" is ignored and never used as the
  output.
