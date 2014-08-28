---
layout: post
title: "Sound Effect Management with an LRU Cache in Java/LibGDX"
description: ""
category: Software
tags: [software,game development,java]
---
{% include JB/setup %}
A least-recently used cache is a particularly useful data structure for managing sound effects on most mobile games.  There are many examples that are easy to find through simple google-fu, but the example I used (<a href="http://stackoverflow.com/questions/224868/easy-simple-to-use-lru-cache-in-java">from StackOverflow</a>) has unfortunately lost its linked source (at least at the time of this writing).

This is the simple LRU cache implementation I use for sound effects (explosion/shooting effects, NOT longer-lived music or background sounds).


{% highlight java linenos=table %}
import java.util.Collection;
import java.util.LinkedHashMap;
import java.util.Map;

/**
 * A simple implementation of an LRU cache.
 * <p>
 * Retrieved from <a href=
 * "http://stackoverflow.com/questions/224868/easy-simple-to-use-lru-cache-in-java"
 * >Stackoverflow</a>.
 *
 * @author Hood, Daniel
 */
public class LRUCache<K, V> {
     /**
      * Called when a cached element is about to be removed.
      */
     public interface CacheEntryRemovedListener<K, V> {
          void notifyEntryRemoved(K key, V value);
     }

     private Map<K, V> cache;
     private CacheEntryRemovedListener<K, V> entryRemovedListener;

     /**
      * Creates the cache with the specified max entries.
      */
     public LRUCache(final int maxEntries) {
          cache = new LinkedHashMap<K, V>((int) Math.ceil(maxEntries * 1.75), .75f, true) {
               public boolean removeEldestEntry(Map.Entry<K, V> eldest) {
                    if (size() > maxEntries) {
                         if (entryRemovedListener != null) {
                              entryRemovedListener.notifyEntryRemoved(
                                        eldest.getKey(), eldest.getValue());
                         }
                         return true;
                    }
                    return false;
               }
          };
     }

     public void add(K key, V value) {
          cache.put(key, value);
     }

     public V get(K key) {
          return cache.get(key);
     }

     public Collection<V> retrieveAll() {
          return cache.values();
     }

     public void setEntryRemovedListener(
               CacheEntryRemovedListener<K, V> entryRemovedListener) {
          this.entryRemovedListener = entryRemovedListener;
     }
}
{% endhighlight %}
Let's go through this line by line for a better understanding.

Line 1-3: Imports.  The LRU cache is backed by Java’s linked hash map, which fortunately for us keeps track of our most recently accessed members.

Line 18-20: Remove listener interface declaration.  An interface that allows us to create any type of custom notification function we can use when an item is removed.  Useful for knowing when to actually dispose of the sound asset (with LibGDX's "dispose" method).

Line 29:  Our cache constructor.  First we construct our new LinkedHashMap specifying three arguments: the initial capacity, the load factor, and whether or not we rate objects by last accessed or last inserted.  Here, we can set the initial capacity to be maxEntries * 1.75.  This setting guarantees we will never have to perform an expensive re-hash while the game is running.  Remember, if the maximum number of entries divided by the load factor is less than the initial capacity, we will never have to rehash.  We don't just want a load factor of 1 because we likely don't have a perfect hash function and thus won't get optimal lookups.  Finally, we specify with a boolean that we want access-ordering (remove the least recently accessed object) as opposed to insertion-ordering.

Line 30-39:  Our entry removal method.  From the <a href="http://docs.oracle.com/javase/7/docs/api/java/util/LinkedHashMap.html#removeEldestEntry">java docs</a> :

>The removeEldestEntry(Map.Entry) method may be overridden to impose a policy for removing stale mappings automatically when new mappings are added to the map.

Here we override the method to remove the oldest (least recently used) entry if we have reached the cache’s max size.  This is where we enforce the max size of the cache and ensure we will never re-hash.  We also trigger the notifyEntryRemoved event.

Line 42+:  Boilerplate.  The rest of the code is just the basic get/set/retrieve operations and a setter for our remove listener.

Now whenever we load/play a sound, we add it/look it up in our cache.  The least-used sounds will be automatically removed from our cache for us (without us having to track what is accessed), and we can easily limit the amount of memory set aside to store sound effects.  Note this implementation isn't thread safe!

**Edit:  Note that you will still want to load all of the necessary sounds for your current screen on the initial load (to avoid stuttering when pulling in new assets).  The advantage of the cache over manual asset management are that (1) Sounds are disposed of automatically when the memory limit is reached and (2) If you have extra space, you can reduce load times between screens (particularly useful if you are switching/loading different screens frequently).
