# NetInterpolation

**NetInterpolation** is a high-performance, zero-allocation temporal interpolation engine for C#. It is designed for high-tick-rate network synchronization in competitive multiplayer environments, featuring adaptive jitter buffering and automatic clock-drift correction.

The interpolator was created for [Project Fika](https://github.com/project-fika).

---

## Key Features

* **Zero-Allocation:** Uses `where T : struct` constraints and `ref readonly` semantics to eliminate GC pressure.
* **Adaptive Jitter Buffer:** Dynamically scales interpolation delay based on real-time network variance using an asymmetric EMA.
* **Clock Synchronization:** Automatically aligns local client time with server time via a smoothed temporal offset.
* **Validation:** Built-in protection against out-of-order, duplicate, or stale UDP packets.
* **Extrapolation:** Supports smooth velocity-based projection to mask micro-stutters during packet loss.

---

## Technical Performance

Benchmarks conducted on .NET Standard 2.1

| Metric | Result |
| :--- | :--- |
| **Memory Allocation** | **0 Bytes** |
| **Insertion Speed** | ~5.8 ns |
| **Sampling Speed** | ~4.2 ns |
| **Algorithm** | $O(\log N)$ Binary Search |

---

## Implementation

### 1. Define Your Data
Implement the `ISnapshot` interface. Ensure your data is a `struct` to take advantage of the memory optimizations.

```csharp
public struct PlayerSnapshot : ISnapshot
{
    // ISnapshot implementation
    public double RemoteTime { get; set; }
    public double LocalTime { get; set; }

    // custom network data
    public Vector3 Position;
    public Vector3 Velocity; // required for high-fidelity extrapolation
    public Quaternion Rotation;

    public PlayerSnapshot(Vector3 pos, Vector3 vel, Quaternion rot, double remote, double local)
    {
        Position = pos;
        Velocity = vel;
        Rotation = rot;
        RemoteTime = remote;
        LocalTime = local;
    }
}
```

### 2. Initialize the Snapshotter
```csharp
private readonly Snapshotter<PlayerSnapshot> _snapshotter = new();
```

---

## Usage

### Adding Data
When a packet arrives, record the **absolute local timestamp** of its arrival. This allows the system to model network jitter accurately.

```csharp
public void OnPacketReceived(PlayerSnapshot newSnapshot)
{
    // Use an absolute timestamp (e.g., Time.unscaledTimeAsDouble)
    newSnapshot.LocalTime = Time.unscaledTimeAsDouble; 
    _snapshotter.AddSnapshot(in newSnapshot);
}
```

### Sampling for rendering
In your `Update()` loop, sample the buffer using the current absolute time. Use `ref readonly` to avoid copying the struct.

```csharp
private void Update()
{
    double now = Time.unscaledTimeAsDouble;
    var state = _snapshotter.GetInterpolationIndices(now, out int from, out int to, out float t);

    if (state == EBufferState.Stale) return;

    // use ref readonly to point directly to buffer memory
    ref readonly var snapFrom = ref _snapshotter.GetSnapshot(from);

    if (state == EBufferState.Interpolating)
    {
        ref readonly var snapTo = ref _snapshotter.GetSnapshot(to);
        
        transform.position = Vector3.LerpUnclamped(snapFrom.Position, snapTo.Position, t);
        transform.rotation = Quaternion.SlerpUnclamped(snapFrom.Rotation, snapTo.Rotation, t);
    }
    else if (state == EBufferState.Extrapolating)
    {
        // project position based on last known velocity to mask jitter
        // 't' is the time elapsed since the last packet
        transform.position = snapFrom.Position + (snapFrom.Velocity * t);
        transform.rotation = snapFrom.Rotation;
    }
}
```

---

## The Buffer States

* **`Interpolating`**: The render time is between two valid snapshots. This is the ideal state for perfect visual sync ($0 \le t \le 1$).
* **`Extrapolating`**: The render time has passed the newest snapshot. The system uses `t` as the delta-time since the last packet to project motion ($t > 0$ as a delta).
* **`Stale`**: The buffer is empty or the gap since the last packet is too wide to guess accurately (e.g., >100ms). The entity should stop moving.

---

## Compatibility
* **.NET Standard 2.1** (Recommended for Unity 2021.2+)
* **.NET Standard 2.0** (Compatible with Unity 2018.x+)