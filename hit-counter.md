# 🚀 Design Hit Counter — Uber SDE-2 (Full End-to-End Answer)

Let me think this through out loud.

We need to design a system that records hits per page and returns how many hits happened in the last 5 minutes.

My goal here is:
- to keep updates O(1)
- keep memory bounded
- and ensure predictable performance even at high traffic

---

## 🟢 Clarification

Before I start designing, I want to clarify a few things:
- Are hits tracked per page or globally?
- Is the window fixed at 5 minutes or configurable?
- Can I assume timestamps are increasing?

(Wait for interviewer response)

---

## 🧠 Brute Force Approach

A simple solution would be:
- store every timestamp for each page
- and during query, filter timestamps within last 5 minutes

But this has issues:
- Query becomes O(n)
- Memory grows unbounded
- Not scalable for high traffic systems

---

## ⚡ Optimized Approach

Instead of storing every hit, I’ll aggregate hits into **fixed-size time buckets**.

### 💡 Key Idea:
- Divide time into units (1 minute)
- Maintain an array of size = 5 (for 5 minutes)
- Each index represents one minute
- Use modulo to reuse array positions (circular buffer)

👉 So instead of storing all hits, we store only counts per minute.

---

## 🏗️ Design

I’ll split the system into:

1. `PageHitCounter` → handles hit counting for a single page  
2. `HitCounterService` → manages multiple pages  

Each page will maintain:
- `buckets[]` → stores hit counts
- `times[]` → stores which minute each bucket belongs to

---

## 💻 Implementation (Detailed Code + Line-by-Line Explanation)

```python
from collections import defaultdict
import time


class PageHitCounter:
    def __init__(self, window=5):
        """
        window: number of minutes we want to track (e.g., 5 minutes)
        """
        
        # Store the window size (e.g., 5 minutes)
        self.window = window
        
        # buckets[i] → number of hits in that minute slot
        # Example: buckets[0] = 10 means 10 hits in that minute
        self.buckets = [0] * window
        
        # times[i] → which minute this bucket represents
        # Example: times[0] = 100 means this bucket stores hits for minute 100
        # Initialize with -1 to indicate empty/uninitialized buckets
        self.times = [-1] * window

    def hit(self, timestamp):
        """
        Record a hit at a given timestamp
        """
        
        # Step 1: Convert timestamp into minute
        # Example: 300 seconds → 5th minute
        minute = timestamp // 60
        
        # Step 2: Map minute to circular index using modulo
        # This ensures we reuse memory (ring buffer)
        idx = minute % self.window

        # Step 3: Check if this bucket is stale
        # If stored minute != current minute → old data → reset
        if self.times[idx] != minute:
            # Update this bucket to current minute
            self.times[idx] = minute
            
            # Reset count because old data is irrelevant now
            self.buckets[idx] = 0

        # Step 4: Increment hit count for this minute
        self.buckets[idx] += 1

    def get_hits(self, timestamp):
        """
        Return number of hits in last 'window' minutes
        """
        
        # Step 1: Convert timestamp to minute
        minute = timestamp // 60
        
        # Step 2: Initialize total hits
        total = 0

        # Step 3: Iterate through all buckets
        for i in range(self.window):
            
            # Check if this bucket is within valid time window
            # Condition: current_minute - stored_minute < window
            # This ensures we only count recent buckets
            if minute - self.times[i] < self.window:
                total += self.buckets[i]

        # Step 4: Return total hits
        return total


class HitCounterService:
    def __init__(self):
        """
        Service layer managing all pages
        """
        
        # Map page → PageHitCounter
        # defaultdict ensures new page automatically gets a counter
        self.pages = defaultdict(PageHitCounter)

    def record_hit(self, page, timestamp=None):
        """
        Record a hit for a specific page
        """
        
        # If timestamp is not provided, use current time
        if timestamp is None:
            timestamp = int(time.time())
        
        # Delegate hit to that page's counter
        self.pages[page].hit(timestamp)

    def get_hits(self, page, timestamp=None):
        """
        Get hit count for a specific page
        """
        
        # If timestamp not provided, use current time
        if timestamp is None:
            timestamp = int(time.time())
        
        # Return hits from page-specific counter
        return self.pages[page].get_hits(timestamp)
