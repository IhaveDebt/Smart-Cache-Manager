// SmartCacheManager.java
// Hybrid LFU/LRU self-tuning in-memory cache demo
// Compile & run: javac SmartCacheManager.java && java SmartCacheManager

import java.util.*;
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

/**
 * SmartCacheManager
 * - Hybrid LFU/LRU cache: tracks frequency and recency
 * - Periodically analyzes access pattern and adjusts policy via weight between LRU and LFU
 * - Simple API: get(key), put(key, value)
 */
public class SmartCacheManager<K, V> {
    private final int capacity;
    private final Map<K, CacheEntry> map = new ConcurrentHashMap<>();
    private final Deque<K> lru = new ConcurrentLinkedDeque<>();
    private final ScheduledExecutorService tuneScheduler = Executors.newSingleThreadScheduledExecutor();
    private volatile double lfuWeight = 0.5; // 0..1, higher => favor LFU
    private final AtomicLong hits = new AtomicLong(), misses = new AtomicLong();

    private class CacheEntry {
        final K key;
        V value;
        final AtomicLong freq = new AtomicLong(0);
        volatile long lastAccess = System.nanoTime();
        CacheEntry(K k, V v) { key = k; value = v; freq.set(1); }
    }

    public SmartCacheManager(int capacity) {
        this.capacity = Math.max(1, capacity);
        // tune policy every 5 seconds
        tuneScheduler.scheduleAtFixedRate(this::tunePolicy, 5, 5, TimeUnit.SECONDS);
    }

    public V get(K key) {
        CacheEntry e = map.get(key);
        if (e == null) {
            misses.incrementAndGet();
            return null;
        }
        e.freq.incrementAndGet();
        e.lastAccess = System.nanoTime();
        // move to front in LRU
        lru.remove(key);
        lru.addFirst(key);
        hits.incrementAndGet();
        return e.value;
    }

    public void put(K key, V value) {
        CacheEntry e = map.get(key);
        if (e != null) {
            e.value = value;
            e.freq.incrementAndGet();
            e.lastAccess = System.nanoTime();
            lru.remove(key);
            lru.addFirst(key);
            return;
        }
        if (map.size() >= capacity) evictOne();
        CacheEntry ne = new CacheEntry(key, value);
        map.put(key, ne);
        lru.addFirst(key);
    }

    private void evictOne() {
        // compute hybrid score for candidates in tail region
        List<K> candidates = new ArrayList<>();
        Iterator<K> it = lru.descendingIterator();
        int sample = Math.min(50, map.size());
        while (it.hasNext() && candidates.size() < sample) {
            candidates.add(it.next());
        }
        // choose key with worst hybrid score
        long now = System.nanoTime();
        K worst = null;
        double worstScore = Double.POSITIVE_INFINITY;
        for (K k : candidates) {
            CacheEntry e = map.get(k);
            if (e == null) continue;
            double recency = (now - e.lastAccess) / 1e9; // seconds
            double freq = e.freq.get();
            // lower score = evict
            double score = lfuWeight * (1.0 / (1 + freq)) + (1 - lfuWeight) * recency;
            if (score < worstScore) { worstScore = score; worst = k; }
        }
        if (worst == null) {
            // fallback LRU tail
            K k = lru.pollLast();
            if (k != null) map.remove(k);
        } else {
            lru.remove(worst);
            map.remove(worst);
        }
    }

    private void tunePolicy() {
        // very simple heuristic: if hit rate decreasing -> favor LFU (increase weight)
        double hitRate = (double) hits.getAndSet(0) / Math.max(1, (hits.get() + misses.getAndSet(0)));
        // since we reset hits but not misses consistently above, use simplified heuristic based on sample
        // Instead collect sample stats: average freqs vs recency
        double avgFreq = map.values().stream().mapToLong(e -> e.freq.get()).average().orElse(1.0);
        double avgRecency = map.values().stream().mapToLong(e -> System.nanoTime() - e.lastAccess).average().orElse(1.0) / 1e9;
        // if avgFreq high (some keys hot), favor LFU; if recency large, favor LRU
        double target = Math.min(0.95, Math.max(0.05, avgFreq / (avgFreq + avgRecency + 1)));
        // smooth update
        lfuWeight = 0.8 * lfuWeight + 0.2 * target;
        System.out.printf("[TUNE] lfuWeight=%.3f avgFreq=%.2f avgRecency=%.2fs size=%d%n", lfuWeight, avgFreq, avgRecency, map.size());
    }

    public void stats() {
        System.out.println("Cache size: " + map.size() + " capacity: " + capacity);
        System.out.println("Hits: " + hits.get() + " Misses: " + misses.get());
    }

    public void shutdown() {
        tuneScheduler.shutdownNow();
    }

    // Demo main
    public static void main(String[] args) throws Exception {
        SmartCacheManager<Integer, String> cache = new SmartCacheManager<>(200);
        Random rnd = new Random();
        ScheduledExecutorService s = Executors.newScheduledThreadPool(4);
        // simulate access patterns: hot keys and long tail
        s.scheduleAtFixedRate(() -> {
            for (int i = 0; i < 100; i++) {
                int key = rnd.nextDouble() < 0.8 ? rnd.nextInt(20) : 20 + rnd.nextInt(200);
                String v = cache.get(key);
                if (v == null) cache.put(key, "val" + key);
            }
        }, 0, 100, TimeUnit.MILLISECONDS);
        // periodic stats
        s.scheduleAtFixedRate(() -> {
            cache.stats();
        }, 2, 5, TimeUnit.SECONDS);
        Thread.sleep(20000);
        s.shutdownNow();
        cache.shutdown();
        System.out.println("Demo finished.");
    }
}
