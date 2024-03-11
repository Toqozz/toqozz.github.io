---
layout: post
title: "Recording Gameplay: The Right Way"
date: 2024-03-04
categories:
---

<video muted="true" autoplay="autoplay" loop="loop" controls="true">
    <source src="/assets/2024_recording_recording.webm" type="video/webm">
</video>
*A recording of a recording.*

Recently I was struggling to record some demo webms on macOS -- basic stuff, like moving a sprite back and forth with some logic.  For some reason, I couldn't get things to move smoothly, without stuttering, so all my recordings were messed up.  This issue itself is bizarre (this should be possible, and trust me, I tried), but that's not the point of this post.  In reality, you're never going to get a truly high quality recording from a screen capture anyway.  Games are constantly fighting the OS for processor time, and even well-made games have frame timing issues.  If you're not pumping out a new frame exactly every 16.6667ms (or whatever your desired framerate is), then your recording is flawed.

The proper solution for recording video is to run your game with a fixed delta time (whatever you want your video framerate to be recorded at) and just spit out images from the game every frame, and then compile those into a video.  You'll get a perfectly smooth recording of your game, regardless of framerate issues.

This is actually how a lot of recording works for video game frag movies.  [Lawena](https://lawena.github.io/) is a tool for source engine games which lets you crank the graphics settings of your game and record at any framerate you want.  It's really important when you're compiling a video like that to have high quality source recordings, sometimes at over 240FPS (for slow-mo effects).

Additionally, when you record directly from the frame buffer, you have control over *when* in the frame your recording should take place.  For example, if you wanted to do some custom post processing, you could copy your frame before post processing passes are applied.

> *Why is my recording flawed if my framerate is slightly off?*
![Recording frames vs game frames.](/assets/2024_recording_frame_timing.png)
Here's a comparison of game frames (white), to recording frames (red diamonds).  Assuming the recording can keep up, when you're recording your screen, you can assume that the recording program is capturing a frame at every interval (60FPS -- ~16.67ms in this case).  Remember that when the recorder captures a frame, it's actually capturing the last displayed frame, which could have been who knows how long ago (the dotted red lines at the top signify how much time we're missing at each recording sample).  The result of this is that if you were to step through the frames in your video file, you'd trivially notice that each frame doesn't have the same amount of movement, despite the recording being at a constant framerate.
>
> I've also seen people, when trying to make recordings where the game won't make framerate, slow the game down to half speed, do the recording, and then speed it up 2x in editing software so that things look smooth.  This is obviously flawed if you understand the above.  Please don't do this.  If you absolutely must, then record in the highest framerate you possibly can so that the 'lost time' is minimized.

### The Naive Solution
The naive solution is obvious.  All you really need is a way to grab the output pixels from the framebuffer, and write those pixels to some kind of image format.  From there you can use other tools like `ffmpeg` to compile the frames into video.

Accessing the framebuffer is not always easy if a suitable abstraction is not provided, but most major engines should have *something*.  I've been meaning to try out [wgpu](https://wgpu.rs/) some more, so I just built something from those primitives.  Here's the basics of how I'm grabbing the output frame pixels and dumping them to a `.png`, for those curious:
```rust
// `end_frame` is a function on a `Recorder` struct which keeps track of recording state etc.
pub fn end_frame(&mut self, device: &wgpu::Device, encoder: &mut wgpu::CommandEncoder, frame_texture: &wgpu::Texture) {
    if !self.recording {
        return;
    }

    // Copy our screen tetxure data into a buffer.
    encoder.copy_texture_to_buffer(
        wgpu::ImageCopyTexture {
            aspect: wgpu::TextureAspect::All,
            texture: frame_texture,
            mip_level: 0,
            origin: wgpu::Origin3d::ZERO,
        },
        wgpu::ImageCopyBuffer {
            buffer: &self.buffer,
            layout: wgpu::ImageDataLayout {
                offset: 0,
                bytes_per_row: Some(4 * self.width),    // 4 bytes (bgra) * width
                rows_per_image: Some(self.height),
            },
        },
        frame_texture.size(),
    );

    // Map the buffer so that we can read it on the CPU side.
    // It's an async operation in wgpu, so just block for simplicity.
    let buffer_slice = self.buffer.slice(..);
    pollster::block_on(async {
        let (tx, rx) = futures_intrusive::channel::shared::oneshot_channel();
        buffer_slice.map_async(wgpu::MapMode::Read, move |result| {
            tx.send(result).unwrap();
        });

        device.poll(wgpu::Maintain::Wait);
        rx.receive().await.unwrap().unwrap();
    });

    // Finally write out our png.
    {
        let filename = format!("frame_{:06}.png", self.frame_number);
        self.frame_number += 1;
        // `data` gets dropped at the end of this scope, so that we can unmap the buffer.
        let data = buffer_slice.get_mapped_range();
        lodepng::encode_file(
            &filename,
            &data,
            self.width as usize,
            self.height as usize,
            lodepng::ColorType::BGRA,
            8,
        ).expect("Failed to write PNG.");
    }

    self.buffer.unmap();
}
```

> It looks like Unity has the [ScreenCapture](https://docs.unity3d.com/ScriptReference/ScreenCapture.html) class for getting pixels, though I suspect there's better ways.

You'll also want to artificially set your game's `dt` to whatever the recording framerate is, so that your game *appears* to be running at that framerate in the recording:
```rust
pub fn update(&mut self, real_dt: f32) {
    let dt = {
        if self.recorder.is_recording() {
            1.0 / self.recorder.recording_framerate() as f32
        } else {
            real_dt
        }
    };

    ...
}
```

You should also consider limiting the game's framerate to match this so that it's actually playable, but note that framerate limiting should not generally be considered a substitute for setting the `dt` if you want a perfect recording.

When you're done, you can use `ffmpeg` to compile the PNGs into whatever format you want:
```sh
ffmpeg -framerate 60 -i frame_%06d.png -c:v libx264 -profile:v high -crf 20 -pix_fmt yuv420p output.mp4
```

This behaves fine, but leaves a lot to be desired.  The first issue is that writing PNGs is *slow*.  It took me 24ms~ on average to write a PNG to disk in a release build, and debug builds are *much* slower.  This alone is enough to stop us from recording and playing at the same time at 60FPS (although the recording itself will still look smooth for reasons that should be clear by now).  The second issue is that this approach quickly eats drive space for longer recordings, especially with complex scenes where PNG's compression isn't doing as much.

![PNG profile capture](/assets/2024_png_profile.png)
*Profile capture taken with [Tracy Profiler](https://github.com/wolfpld/tracy).*

> `lodepng` is not the best we can do.  [There are faster PNG encoders](https://github.com/richgel999/fpng).  We could also turn down the compression for a faster write, or [use a different format altogether](https://blender.stackexchange.com/questions/148231/what-image-format-encodes-the-fastest-or-at-least-faster-png-is-too-slow).

> If you're wondering how much of this time is the OS writing the file to disk: it's about a millisecond faster on average when we encode into memory.

### Encoding With `ffmpeg` Directly
It turns out that `ffmpeg` actually accepts raw frames (of course).  So you can simplify this whole pipeline by just sending the raw data to `ffmpeg` directly and letting it handle it.  Here's roughly the process I'm using:
- When recording starts:
    - Launch an `ffmpeg` process in a separate thread.  Spin in that thread, writing all the received frames into the process' `stdin`.
    - Enable 'recording mode', which sets the game's `dt` to `1.0/60.0` (my target recording framerate) and limits the game's framerate so that things are playable.
- At the end of every frame, grab the frame output and send the raw bytes through to the `ffmpeg` thread.  You'll want to buffer (i.e. copy) the output here, since `ffmpeg` might take longer than a frame to encode your video frame.
- When recording stops:
    - Close `stdin` on the `ffmpeg` thread and `.join()` from the main thread (this is blocking until `ffmpeg` finishes writing the file).
    - Disable recording mode.
    
The `ffmpeg` command you'll want for raw frames is something like the following:
```sh
ffmpeg
    -r 60               # your framerate
    -f rawvideo
    -vcodec rawvideo
    -pix_fmt bgra       # your pixel format
    -s 1280x720         # your window size
    -i -                # input as stdin
    -c:v libvpx-vp9     # standard webm codec
    -pix_fmt yuv420p    # standard pixel format
    output.webm
```
    
There's really not much to it.  In fact, it almost doesn't feel like it's worth writing about.  But before I got this working I didn't really have a concrete idea about how this all should work, so I figured it's worth sharing.

> You'll notice that I'm sidestepping my 'first' issue with writing the PNGs: encoding with `ffmpeg` is also slow, and we can't do it on the main thread.  You could also encode your PNGs in a separate thread to the same effect, or just use a format that's faster to encode.  I mostly just wanted to complain and show off the Tracy Profiler.  I guess I'm also more OK with accepting a separate thread for handling the whole recording/encoding pipeline, rather than just for writing out a buffer of images.

---

Here's some code for how I'm doing my recordings.  The main thing I'm yet to do is wait for the `ffmpeg` process to finish in a non-blocking way, and display the progress (displaying the frames waiting for encoding would be enough).  This code is truncated to make it easier to read for the purposes of the post; you can find the full source on [Github](https://github.com/Toqozz/blog-code/blob/master/recording/recorder.rs).

```rust
// recorder.rs
pub fn begin_record(&mut self) {
    self.recording = true;       

    assert!(self.ffmpeg.is_none());
    let ffmpeg = {
        let (send, recv) = mpsc::channel::<Vec<u8>>();
        let (width, height) = (self.width, self.height);
        let handle = thread::spawn(move || {
            // do ffmpeg stuff
            let size = format!("{}x{}", width, height);
            let ffmpeg_cmd = "ffmpeg";
            let args = [
                "-r", "60",                     // Input frame rate.
                "-f", "rawvideo",               // Input format.
                "-vcodec", "rawvideo",
                "-pix_fmt", "bgra",
                "-s", &size,
                "-i", "-",                      // Input as stdin.
                "-c:v", "libvpx-vp9",           // Output codec.
                "-pix_fmt", "yuva420p",
                "-y",                           // Overwrite output file.
                "output.webm",                  // File name.
            ];

            let mut child = std::process::Command::new(ffmpeg_cmd)
                .args(&args)
                .stdin(Stdio::piped())
                .spawn()
                .expect("Failed to spawn FFmpeg command");
            
            let stdin = child.stdin.as_mut().expect("Couldn't open ffmpeg stdin.");
            
            for data in recv {
                stdin.write_all(data.as_slice()).unwrap();
            }
            
            let output = child.wait_with_output().unwrap();
            // Check the output and error messages
            if output.status.success() {
                println!("FFmpeg command executed successfully.");
            } else {
                // The `stderr` field of the output contains any error messages
                let error_message = String::from_utf8_lossy(&output.stderr);
                println!("FFmpeg command failed: {}", error_message);
            }
        });
        
        FfmpegProcess {
            send,
            handle,
        }
    };
    
    self.ffmpeg = Some(ffmpeg);
}

pub fn end_frame(
    &mut self,
    device: &wgpu::Device,
    encoder: &mut wgpu::CommandEncoder,
    frame_texture: &wgpu::Texture
) {
    if !self.recording {
        return;
    }

    encoder.copy_texture_to_buffer(
        wgpu::ImageCopyTexture {
            aspect: wgpu::TextureAspect::All,
            texture: frame_texture,
            mip_level: 0,
            origin: wgpu::Origin3d::ZERO,
        },
        wgpu::ImageCopyBuffer {
            buffer: &self.buffer,
            layout: wgpu::ImageDataLayout {
                offset: 0,
                bytes_per_row: Some(4 * self.width),    // 4 bytes (bgra) * width
                rows_per_image: Some(self.height),
            },
        },
        frame_texture.size(),
    );

    let buffer_slice = self.buffer.slice(..);
    pollster::block_on(async {
        let (tx, rx) = futures_intrusive::channel::shared::oneshot_channel();
        buffer_slice.map_async(wgpu::MapMode::Read, move |result| {
            tx.send(result).unwrap();
        });
        device.poll(wgpu::Maintain::Wait);
        rx.receive().await.unwrap().unwrap();
    });
            
    {
        // We need to drop this before unmapping.
        let data = buffer_slice.get_mapped_range();

        // Send buffer to recording thread.  We need to copy the data to do this safely.
        if self.recording {
            let ffmpeg = self.ffmpeg.as_mut().unwrap();
            let mut vec = Vec::with_capacity(data.len());
            vec.extend_from_slice(&data);
            ffmpeg.send.send(v).expect("Failed to send to ffmpeg thread.");
        }
    }

    self.buffer.unmap();
}

pub fn end_record(&mut self) {
    self.recording = false;
    
    let ffmpeg = self.ffmpeg.take().unwrap();
    drop(ffmpeg.send);
    ffmpeg.handle.join().unwrap();
}
```

*[Discuss on GitHub](https://github.com/Toqozz/blog-code/issues/10)*