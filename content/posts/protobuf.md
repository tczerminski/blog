+++
authors = ["Tomasz Czermiński"]
title = "Beating Go’s Default Protobuf for Prometheus Remote Write"
date = "2025-09-27"
description = "Exploring why Go's default protobuf unmarshal slows down Prometheus Remote Write, and how a custom parser doubled throughput."
tags = [
    "performance",
    "go",
]
categories = [
]
series = ["Performance"]
aliases = []
+++

# Beating Go’s Default Protobuf for Prometheus Remote Write

## Project Background

The story starts with an observability platform we were building for a large European e-commerce.  
It handled metrics scraping, monitoring checks, and alerting — though not logs or tracing.

As adoption grew, so did the problems:

- Some teams exported metrics with **UUIDs as label values**.  
- Others had pods stuck in **CrashLoopBack**, generating a fresh set of labels on each restart.  
- We had **no way to impose restrictions** on clients due to internal business politics.  

The result was **high-cardinality explosions** that caused several outages.

At the time, we were using **VictoriaMetrics standalone**, which had no concept of multi-tenancy:  
all timeseries went into a single blob. That meant **one misbehaving user could bring down everyone**.

We considered moving to **VictoriaMetrics Cluster**, but key features like downsampling were behind a paywall.  
When we asked about enterprise pricing, the quote was so far above our budget it wasn’t even close.

So we turned to **Cortex**. As a CNCF project, it’s fully open source and supports true multi-tenancy.  
The plan was simple: distribute metrics among tenants by populating the `X-Scope-OrgID` HTTP header in the Prometheus Remote Write protocol.  
The value for this header came from the `service_id` label, which every exported metric was required to define.

The final architecture looked like this:

- Prometheus scrapers collect metrics.  
- VMAgent does light transformations.  
- Cortex Distributor stores each timeseries in the correct tenant.  

The missing piece was a **relay**: a lightweight service that intercepted Remote Write traffic between VMAgent and Cortex, grouped metrics by `service_id`, and added the proper HTTP header.

That’s when the real challenge began — building a relay fast enough to handle **gigabytes per second** of traffic.  
And the very first bottleneck turned out to be… Go’s default protobuf unmarshaller.


## The code

```go

type Algo string

const (
    SNAPPY Algo = "snappy"
    ZSTD Algo = "zstd"
)

func decompress(algo Algo, payload []byte) (decompressed []byte, release func(), err error) {
	buffer := buffersPool.Get().([]byte)
	switch algo {
	case SNAPPY:
		decompressed, err = snappy.Decode(buffer, payload)
		release = func() {
			if cap(decompressed) > maxCap {
				return
			}
			buffersPool.Put(decompressed[:0])
		}
	case ZSTD:
		dec := decodersPool.Get().(*zstd.Decoder)
		decompressed, err = dec.DecodeAll(payload, buffer)
		decodersPool.Put(dec)
		release = func() {
			if cap(decompressed) > maxCap {
				return
			}
			buffersPool.Put(decompressed[:0])
		}
	default:
		err = fmt.Errorf("%w: %s", ErrUnsupportedEncoding, algo)
	}
	return decompressed, release, err
}

func Dispatch(algo string, compressed []byte) error {
	payload, releaseBuffer, err := decompress(algo, compressed)
	if err != nil {
		return err
	}
	defer releaseBuffer()
	tss, releaseTss, err := Unmarshal(payload)
	if err != nil {
		return err
	}
    // We would extract service_id label here
}

func UnsafeString(b []byte) string {
	// #nosec G103
	return *(*string)(unsafe.Pointer(&b))
}

func v1() func(*fasthttp.RequestCtx) {
	return func(ctx *fasthttp.RequestCtx) {
		algo := UnsafeString(ctx.Request.Header.Peek("Content-Encoding"))
		if err := Dispatch(algo, ctx.Request.Body()); err != nil {
			ctx.Error(err.Error(), fasthttp.StatusBadGateway)
			return
		}
		if err := ctx.Request.CloseBodyStream(); err != nil {
			ctx.SetStatusCode(fasthttp.StatusInternalServerError)
			return
		}
		ctx.SetStatusCode(fasthttp.StatusOK)
	}
}
```

The critical point in the code above is that everything was designed with performance in mind.
We chose fasthttp, so we have HTTP layer, meaning parsing and socket handling, already done. But in order to really utilize it, runtime allocations must be minimized.
That means, using *sync.Pool*, reusing slices and keeping track of lifetimes, as reference to request or its derivative must not be kept after it is released.

This post is about protobuf, but I want to emphasize: everything else in the relay was tuned for performance. The only real bottleneck left was the default Go protobuf unmarshaller.


## Benchmarking the defaults

I have created a following benchmark:

```go
b.ResetTimer()
for i := 0; i < b.N; i++ {
    if err := Dispatch(ZSTD, payload); err != nil {
        b.Fatal(err)
    }
}
```

and got following results:
```
goos: darwin
goarch: arm64
pkg: relay/tests
cpu: Apple
Benchmark_Dispatch-12               8372            127002 ns/op          193759 B/op        268 allocs/op
PASS
ok      relay/tests      2.127s
```

Let us take a look into pprof:
```
Type: alloc_space
Time: 2025-09-29 11:33:09 CEST
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 1583.66MB, 98.75% of 1603.76MB total
Dropped 68 nodes (cum <= 8.02MB)
Showing top 10 nodes out of 24
      flat  flat%   sum%        cum   cum%
 1202.18MB 74.96% 74.96%  1203.18MB 75.02%  github.com/prometheus/prometheus/prompb.(*TimeSeries).Unmarshal
  169.68MB 10.58% 85.54%   169.68MB 10.58%  github.com/gogo/protobuf/proto.Marshal
   61.69MB  3.85% 89.39%  1264.86MB 78.87%  github.com/prometheus/prometheus/prompb.(*WriteRequest).Unmarshal
      52MB  3.24% 92.63%       52MB  3.24%  github.com/klauspost/compress/zstd.encoderOptions.encoder
   46.52MB  2.90% 95.53%  1539.17MB 95.97%  relay/pkg.Dispatch
   26.97MB  1.68% 97.21%    26.97MB  1.68%  github.com/klauspost/compress/zstd.(*blockDec).reset
   23.12MB  1.44% 98.65%    23.12MB  1.44%  github.com/klauspost/compress/zstd.init.func2
       1MB 0.062% 98.71%  1596.57MB 99.55%  tests.Benchmark_Dispatch
    0.51MB 0.032% 98.75%    52.10MB  3.25%  github.com/klauspost/compress/zstd.(*Decoder).DecodeAll
         0     0% 98.75%    56.10MB  3.50%  relay/pkg.decompress
```

### Why the default unmarshaller hurts

The profile is pretty unambiguous:  
over **75% of all allocations** come from `prompb.TimeSeries.Unmarshal`.  

That’s not too surprising when you think about it:  
the default Go protobuf implementation eagerly allocates Go structs for every label, sample, and timeseries.  
Even if you only need a single label like `service_id`, you still pay the cost of fully materializing the entire `WriteRequest`.

For us, this meant:

- Hundreds of allocations per request.  
- Gigabytes of heap churn per second at scale.  
- GC pressure that quickly became the bottleneck.

The protobuf layer was drowning everything else.

---

## A few words about Protobuf

The crucial thing to understand about Protobuf is that it’s a **generic serialization format**.  
It’s designed to support arbitrary data structures: you define fields, tag them with IDs, specify types, and later you can extend the schema by adding new fields.  
But you pay for this flexibility with **reflection and allocations**.  

In most cases, this is fantastic. Protobuf gives you a compact binary wire format, backward compatibility, and performance that’s usually much better than JSON RPC or REST.  

And to be fair — there are really fast Go protobuf libraries out there.  
They avoid reflection, use code generation, and can be significantly faster than the defaults.  
But they’re still **generic decoders**, and without arenas they can’t avoid the heap churn when you don’t need the full structure.  

In my case, that generic approach was working against me.  
For Prometheus Remote Write, I didn’t need the full flexibility of Protobuf.  
I only needed to extract a single label (`service_id`) out of the payload.  

In other words: I didn’t need a general-purpose decoder.  
I needed a **tailored parser**, specific to Remote Write 1.0.  

If I had been using C or C++, this wouldn’t even be a problem.  
Their protobuf libraries support **memory arenas**, which sidestep these allocation issues entirely.  

But in Go, there’s no runtime support for arenas — and likely never will be.  
That left me with one option: build a tailored parser myself.

## What about arenas in Go? (Proposal #51317)

While Go doesn’t currently support runtime memory arenas, there *is* an open proposal in the Go project: issue **#51317**, titled *“proposal: arena: new package providing memory arenas”*.  

The idea is to allow allocations into a contiguous arena and then free them all at once. For systems like ours—where many allocations are short-lived per request—this could, in theory, reduce GC overhead significantly.  

However, the proposal is **on hold indefinitely** due to serious API and safety concerns. And beyond the technical hurdles, arenas also run somewhat against Go’s design philosophy. One of Go’s strengths is its simplicity and predictable memory model; introducing arenas would complicate both the mental model and the runtime.  

## Unmarshaling `WriteRequest` – fast and cheap

Before jumping into code, it helps to understand a few low-level Protobuf details:

* **Tags are integers**
  Every field in a message has a unique tag number. Once assigned, it cannot be reused—this guarantees backward compatibility.

* **Wire type + tag = field header**
  Each field begins with a small header containing its tag and wire type.
  The wire type tells us how to parse the value that follows (varint, fixed64, length-delimited, etc.).

* **Varints save space**
  Integers are encoded with variable-length encoding. Small numbers take fewer bytes.
  (This isn’t unique to Protobuf—Go even provides `binary.Uvarint` in its standard library.)

* **Object pooling is critical**
  To avoid constant allocations, I added a pool for reusing `TimeSeries` structures.
  This cut down both GC pressure and latency spikes.

---

### The `WriteRequest` unmarshaller

With those building blocks in place, here’s the tailored unmarshaller for `WriteRequest`:

```go
func Unmarshal(buf []byte) (tss []*TS, release func(), err error) {
    release = func() {
        for _, ts := range tss {
            Put(ts) // return to pool
        }
    }

    for len(buf) > 0 {
        key, n := binary.Uvarint(buf)
        buf = buf[n:]
        fieldNum := key >> 3
        wireType := key & 0b111

        if fieldNum != 1 || wireType != 2 {
            return tss, release, fmt.Errorf("unexpected field/wiretype")
        }

        tsLen, n := binary.Uvarint(buf)
        buf = buf[n:]
        tsBuf := buf[:tsLen]
        buf = buf[tsLen:]

        ts := Get() // grab from pool
        if err := timeseries(&ts.TimeSeries, tsBuf); err != nil {
            return nil, release, err
        }
        tss = append(tss, ts)
    }

    return tss, release, nil
}
```

We only care about **time series**. Exemplars and other fields are ignored.
That’s why any unrecognized field is just skipped.

---

### Decoding a `TimeSeries`

Each `TimeSeries` contains labels and samples.
Here’s how I handle it:

```go
func timeseries(ts *prompb.TimeSeries, buf []byte) error {
    ts.Labels = ts.Labels[:0]
    ts.Samples = ts.Samples[:0]

    for len(buf) > 0 {
        key, n := binary.Uvarint(buf)
        if n <= 0 {
            return errors.New("failed to read key")
        }
        buf = buf[n:]
        fieldNum := key >> 3
        wireType := key & 0b111

        switch fieldNum {
        case 1: // labels
            if wireType != 2 {
                return fmt.Errorf("unexpected wire type for labels: %d", wireType)
            }
            lblLen, n := binary.Uvarint(buf)
            if n <= 0 || int(lblLen) > len(buf[n:]) {
                return errors.New("invalid label length")
            }
            lblBuf := buf[n : n+int(lblLen)]
            buf = buf[n+int(lblLen):]
            var lbl prompb.Label
            if err := label(&lbl, lblBuf); err != nil {
                return err
            }
            ts.Labels = append(ts.Labels, lbl)

        case 2: // samples
            if wireType != 2 {
                return fmt.Errorf("unexpected wire type for samples: %d", wireType)
            }
            sLen, n := binary.Uvarint(buf)
            if n <= 0 || int(sLen) > len(buf[n:]) {
                return errors.New("invalid sample length")
            }
            sBuf := buf[n : n+int(sLen)]
            buf = buf[n+int(sLen):]
            var s prompb.Sample
            if err := sample(&s, sBuf); err != nil {
                return err
            }
            ts.Samples = append(ts.Samples, s)

        default: // skip unknown fields
            switch wireType {
            case 0: // varint
                _, n := binary.Uvarint(buf)
                if n <= 0 {
                    return errors.New("invalid varint skip")
                }
                buf = buf[n:]
            case 1: // fixed64
                buf = buf[8:]
            case 2: // length-delimited
                l, n := binary.Uvarint(buf)
                buf = buf[n+int(l):]
            case 5: // fixed32
                buf = buf[4:]
            default:
                return fmt.Errorf("unsupported wire type: %d", wireType)
            }
        }
    }
    return nil
}
```

The idea is simple:
iterate over the buffer until it’s empty, slice it along the way, and decode only the fields we care about.

---

### Decoding labels and samples

Both are straightforward:

```go
func label(lbl *prompb.Label, buf []byte) error {
    for len(buf) > 0 {
        key, n := binary.Uvarint(buf)
        if n <= 0 {
            return errors.New("failed to read label key")
        }
        buf = buf[n:]
        fieldNum := key >> 3
        wireType := key & 0b111
        if wireType != 2 {
            return fmt.Errorf("unexpected wire type for label field: %d", wireType)
        }
        strLen, n := binary.Uvarint(buf)
        if n <= 0 || int(strLen) > len(buf[n:]) {
            return errors.New("invalid string length in label")
        }
        strBuf := buf[n : n+int(strLen)]
        buf = buf[n+int(strLen):]
        switch fieldNum {
        case 1:
            lbl.Name = string(strBuf)
        case 2:
            lbl.Value = string(strBuf)
        }
    }
    return nil
}

func sample(s *prompb.Sample, buf []byte) error {
    for len(buf) > 0 {
        key, n := binary.Uvarint(buf)
        if n <= 0 {
            return errors.New("failed to read sample key")
        }
        buf = buf[n:]
        fieldNum := key >> 3
        wireType := key & 0b111

        switch fieldNum {
        case 1: // value = double = fixed64
            if wireType != 1 || len(buf) < 8 {
                return errors.New("invalid double value")
            }
            bits := binary.LittleEndian.Uint64(buf[:8])
            s.Value = math.Float64frombits(bits)
            buf = buf[8:]

        case 2: // timestamp = varint
            if wireType != 0 {
                return errors.New("invalid wire type for timestamp")
            }
            ts, n := binary.Uvarint(buf)
            if n <= 0 {
                return errors.New("invalid timestamp varint")
            }
            s.Timestamp = int64(ts)
            buf = buf[n:]

        default: // skip unknown fields
            switch wireType {
            case 0:
                _, n := binary.Uvarint(buf)
                buf = buf[n:]
            case 1:
                buf = buf[8:]
            case 2:
                l, n := binary.Uvarint(buf)
                buf = buf[n+int(l):]
            case 5:
                buf = buf[4:]
            default:
                return fmt.Errorf("unsupported wire type: %d", wireType)
            }
        }
    }
    return nil
}
```

---

👉 The bottom line:
By focusing only on `TimeSeries` (labels + samples), skipping everything else, and reusing objects from a pool, the unmarshaller becomes **fast, predictable, and memory-efficient**.
Perfect for Remote Write ingestion at high scale.

## Benchmarking my unmarshaller

Once again, let’s benchmark the code — this time replacing the default Protobuf implementation with a custom unmarshaller. The benchmark itself remains unchanged, which allows for a fair, apples‑to‑apples comparison.

### Benchmark output

```
goos: darwin
goarch: arm64
pkg: relay/tests
cpu: Apple
Benchmark_Dispatch-12              11284            104857 ns/op           28232 B/op        111 allocs/op
PASS
ok      relay/tests      2.655s
```

Compared to the earlier results with the default Protobuf library, this shows a **noticeable drop in allocations** as well as **lower execution time** per operation. While the numbers might seem small in isolation, in a high-throughput system such as mine (≈4 Gbps), they add up quickly. Every avoided allocation reduces garbage collector workload, which in turn lowers pause frequency and stabilizes latency — especially at the tail (p99/p999). The freed CPU cycles can then be spent where they matter more, like serialization and compression. Beyond immediate gains, this also provides valuable headroom for scaling without hitting performance cliffs as load increases.  


### Memory profile (pprof)

```
Type: alloc_space
Time: 2025-09-29 17:53:57 CEST
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) top10
Showing nodes accounting for 557.43MB, 98.06% of 568.47MB total
Dropped 41 nodes (cum <= 2.84MB)
Showing top 10 nodes out of 29
      flat  flat%   sum%        cum   cum%
  347.34MB 61.10% 61.10%   347.34MB 61.10%  github.com/gogo/protobuf/proto.Marshal
  107.05MB 18.83% 79.93%   494.32MB 86.95%  relay/pkg.Dispatch
   62.95MB 11.07% 91.00%    62.95MB 11.07%  github.com/klauspost/compress/zstd.encoderOptions.encoder
   11.50MB  2.02% 93.03%    19.21MB  3.38%  relay/pkg.Unmarshal
   11.19MB  1.97% 95.00%    11.19MB  1.97%  github.com/klauspost/compress/zstd.(*blockDec).reset
    5.70MB  1.00% 96.00%     5.70MB  1.00%  relay/pkg.init.func3
    4.47MB  0.79% 96.79%     4.47MB  0.79%  github.com/klauspost/compress/zstd.(*blockEnc).init
    4.02MB  0.71% 97.49%     4.02MB  0.71%  github.com/klauspost/compress/zstd.init.func2
    1.70MB   0.3% 97.79%     6.69MB  1.18%  github.com/klauspost/compress/zstd.(*Encoder).Reset
    1.50MB  0.26% 98.06%    18.22MB  3.20%  relay/pkg.decompress
```

### Observations

* **Deserialization is no longer a hotspot.** In the previous profile, deserialization was eating a large chunk of allocations. With the tailored unmarshaller, its footprint is down to just ~2%.
* **`proto.Marshal` still dominates.** The serialization path remains the single biggest contributor to allocation pressure. This makes sense — Remote Write still requires encoding back into protobuf wire format when shipping samples, and the encoder is still generic.
* **Compression is non‑trivial.** Zstd compression shows up prominently in the profile. While not the focus of this experiment, it’s a reminder that in end‑to‑end pipelines, serialization and compression overheads often dominate CPU and memory.

### Key takeaway

Optimizing Protobuf deserialization gave us a **leaner, lower‑allocation hot path**, directly improving throughput and latency stability. However, the profile also reveals the next bottleneck: serialization. In other words, *every performance win just shifts the spotlight to the next limiting factor* — and that’s exactly how profiling‑driven optimization should work.

## Final thoughts

Generic libraries are great — they solve a wide range of problems with one tool. But when you’re pushing systems at multi-gigabit throughput, the overhead of generality becomes very real.  

In my case, Protobuf’s default Go implementation was doing far more work than I needed. By focusing on a **narrow use case** (extracting a single label from Remote Write payloads) and writing a **custom unmarshaller**, I was able to cut allocations, lower latency, and make the system more predictable under load.  

The takeaway is simple:  
- Always measure before optimizing.  
- Let profiling show you where the real bottlenecks are.  
- Don’t be afraid to go beyond “default” libraries when your workload demands it.  

One thing I still don’t quite understand is why Remote Write relies on such a **generic binary format**. Metrics are a very specific workload, and they could benefit from a format tailored to their characteristics. The overhead of serialization alone, at least in large organizations, directly translates into **significant infrastructure costs.**.  
