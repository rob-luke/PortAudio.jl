PortAudio.jl
============

[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](https://JuliaAudio.github.io/PortAudio.jl/dev)
[![Tests](https://github.com/JuliaAudio/PortAudio.jl/actions/workflows/Tests.yml/badge.svg)](https://github.com/JuliaAudio/PortAudio.jl/actions/workflows/Tests.yml)
[![codecov](https://codecov.io/gh/JuliaAudio/PortAudio.jl/branch/master/graph/badge.svg?token=mgDAi8ulPY)](https://codecov.io/gh/JuliaAudio/PortAudio.jl)


PortAudio.jl is a wrapper for [libportaudio](http://www.portaudio.com/), which gives cross-platform access to audio devices. It is compatible with the types defined in [SampledSignals.jl](https://github.com/JuliaAudio/SampledSignals.jl). It provides a `PortAudioStream` type, which can be read from and written to.

## Opening a stream

The easiest way to open a source or sink is with the default `PortAudioStream()` constructor, which will open a 2-in, 2-out stream to your system's default device(s). The constructor can also take the input and output channel counts as positional arguments, or a variety of other keyword arguments.

```julia
PortAudioStream(inchans=2, outchans=2; eltype=Float32, samplerate=48000Hz, latency=0.1, synced=false)
```

You can open a specific device by adding it as the first argument, either as a `PortAudioDevice` instance or by name. You can also give separate names or devices if you want different input and output devices

```julia
PortAudioStream(device::PortAudioDevice, args...; kwargs...)
PortAudioStream(devname::AbstractString, args...; kwargs...)
```

You can get a list of your system's devices with the `PortAudio.devices()` function:

```julia
julia> PortAudio.devices()
6-element Array{PortAudio.PortAudioDevice,1}:
 PortAudio.PortAudioDevice("AirPlay","Core Audio",0,2,0)
 PortAudio.PortAudioDevice("Built-in Microph","Core Audio",2,0,1)
 PortAudio.PortAudioDevice("Built-in Output","Core Audio",0,2,2)
 PortAudio.PortAudioDevice("JackRouter","Core Audio",2,2,3)
 PortAudio.PortAudioDevice("After Effects 13.5","Core Audio",0,0,4)
 PortAudio.PortAudioDevice("Built-In Aggregate","Core Audio",2,2,5)
```

### Input/Output Synchronization

The `synced` keyword argument to `PortAudioStream` controls whether the input and output ringbuffers are kept synchronized or not, which only effects duplex streams. It should be set to `true` if you need consistent input-to-output latency. In a synchronized stream, the underlying PortAudio callback will only read and write to the buffers an equal number of frames. In a synchronized stream, the user must also read and write an equal number of frames to the stream. If it is only written to or read from, it will eventually block. This is why it is `false` by default.


## Reading and Writing

The `PortAudioStream` type has `source` and `sink` fields which are of type `PortAudioSource <: SampleSource` and `PortAudioSink <: SampleSink`, respectively. are subtypes of `SampleSource` and `SampleSink`, respectively (from [SampledSignals.jl](https://github.com/JuliaAudio/SampledSignals.jl)). This means they support all the stream and buffer features defined there. For example, if you load SampledSignals with `using SampledSignals` you can read 5 seconds to a buffer with `buf = read(stream.source, 5s)`, regardless of the sample rate of the device.

PortAudio.jl also provides convenience wrappers around the `PortAudioStream` type so you can read and write to it directly, e.g. `write(stream, stream)` will set up a loopback that will read from the input and play it back on the output.

## Debugging

If you are experiencing issues and wish to view detailed logging and debug information, set

```
ENV["JULIA_DEBUG"] = :PortAudio
```

before using the package.

## Examples

### Set up an audio pass-through from microphone to speaker

```julia
stream = PortAudioStream(2, 2)
try
    # cancel with Ctrl-C
    write(stream, stream)
finally
    close(stream)
end
```

### Use `do` syntax to auto-close the stream
```julia
PortAudioStream(2, 2) do stream
    write(stream, stream)
end
```

### Open your built-in microphone and speaker by name
```julia
PortAudioStream("Built-in Microph", "Built-in Output") do stream
    write(stream, stream)
end
```

### Record 10 seconds of audio and save to an ogg file

```julia
julia> using PortAudio, SampledSignals, LibSndFile

julia> stream = PortAudioStream("Built-in Microph", 2, 0)
PortAudio.PortAudioStream{Float32,SIUnits.SIQuantity{Int64,0,0,-1,0,0,0,0,0,0}}
  Samplerate: 48000 s⁻¹
  Buffer Size: 4096 frames
  2 channel source: "Built-in Microph"

julia> buf = read(stream, 10s)
480000-frame, 2-channel SampleBuf{Float32, 2, SIUnits.SIQuantity{Int64,0,0,-1,0,0,0,0,0,0}}
10.0 s at 48000 s⁻¹
▁▄▂▃▅▃▂▄▃▂▂▁▁▂▂▁▁▄▃▁▁▄▂▁▁▁▄▃▁▁▃▃▁▁▁▁▁▁▁▁▄▄▄▄▄▂▂▂▁▃▃▁▃▄▂▁▁▁▁▃▃▂▁▁▁▁▁▁▃▃▂▂▁▃▃▃▁▁▁▁
▁▄▂▃▅▃▂▄▃▂▂▁▁▂▂▁▁▄▃▁▁▄▂▁▁▁▄▃▁▁▃▃▁▁▁▁▁▁▁▁▄▄▄▄▄▂▂▂▁▃▃▁▃▄▂▁▁▁▁▃▃▂▁▁▁▁▁▁▃▃▂▂▁▃▃▃▁▁▁▁

julia> close(stream)

julia> save(joinpath(homedir(), "Desktop", "myvoice.ogg"), buf)
