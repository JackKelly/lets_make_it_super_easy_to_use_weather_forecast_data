**Let's make it super easy to use weather forecast data**

My current obsession is to make it as easy as possible to train, run, and research energy forecasting systems.
We're a long way from this "dream" right now! But we're getting there! Please see [this blog post for more info on this idea](https://openclimatefix.org/post/lazy-loading-making-it-easier-to-access-vast-datasets-of-weather-satellite-data).

# Use cases and benefits:
- Academics and students can easily experiment with energy forecasting (using realistic datasets).
- Companies can build state-of-the-art energy forecasts in-house (e.g. grid operators, battery optimisers, etc.).
- We can _finally_ build a public leaderboard of different energy forecasting algorithms.
- Maybe we could run a simple and cheap energy forecasting service with state-of-the-art performance.

# Broad roadmap:
1. Make it trivial to lazily open petabyte-sized numerical weather datasets. To this end, I'm currently working on [`hypergrib`](https://github.com/jackkelly/hypergrib) and [`explore_nwps`](https://github.com/JackKelly/explore_nwps).
2. But some read patterns will never be well-served by reading directly from GRIB. This is because each GRIB message extends across the entire horizontal geospatial extent of the dataset so it will be inefficient to read data for a small number of geospatial locations from GRIB. So the next project I plan to work on will be an automatic caching system. Again, the focus will be on making life as easy as possible for the developer. See below for more details.

Once these essential components are in place, some other cool projects might include (in no particular order):
- An analysis tool (with a UI a bit like [Windy.com](https://windy.com)'s UI?) for comparing different NWPs against each other and against ground truth. 
- On the fly processing and analytics. E.g. reprojection.
- Distribute `hypergrib`'s workload across multiple machines. So, for example, users can get acceptable IO performance even if they ask for "churro-shaped" data arrays.
- Build a public leaderboard of different energy forecasting algorithms, measured against a standard validation dataset (again, using the types of huge datasets that are used in industry, rather than toy academic datasets). Anyone could contribute algorithms to the leaderboard.
- Build ML models using all this lovely data! :)

# Details

## Tool for comparing NWPs against each other, and against ground truth
Where would `hypergrib` run? Perhaps _in_ the browser, using `wasm`?! (but [tokio's `rt-multi-thread` feature doesn't work on `wasm`](https://docs.rs/tokio_wasi/latest/tokio/#wasm-support), which might be a deal-breaker.) Or perhaps run a web service in the cloud, close to the data, across multiple machines. And [expose a standards compliant API like Environmental Data Retrieval](https://github.com/JackKelly/hypergrib/issues/19) for the front-end?

Perhaps include tools to look at correlation between NWPs and actual measurements, including actual measurements from the energy system (like wind power production or solar power generation).

## Caching GRIB data so you still get high performance for read patterns which don't fit with GRIB's data layout
What if users want to read long timeseries for a small number of geographical points (a "churro-shaped" array)? No amount of `hypergrib` trickery will get round the physical constraint that each GRIB message is a 2D array representing a horizontal plane.

So we'll need a way to cache these "churro-shaped" arrays.

Perhaps the "dream" would be to have a completely transparent caching layer. The user would just say "I want data in this shape; and my read pattern will look like this" and the software transparently creates a high-speed cache of that data (in cloud object storage or local disks). Perhaps using the `Icechunk` storage engine or `zarrs`. It must also be very easy to update the cached dataset with new NWP data (e.g. so the same cached dataset can be used for ML training and for inference.).

One idea would be to write a Rust command-line app which creates Zarrs from `hypergrib` data. e.g. "Get data over the United Kingdom from 2017 to today from these three NWPs, and save to a Zarr with this chunk shape, and distribute the workload across 8 VMs". And, later, make it easy to update the Zarr with recent data.

But what if users want to perform arbitrary processing on the data as it's being moved from GRIB to Zarr? Can we optimise & orchestrate exposing the GRIB data to Python, running Python processing, and save to Zarr? Perhaps, at least to start with, we say that it's not possible to perform any computation as the data is being copied to a local cache (other than lossless compression, including reducing the numerical precision. For example, we could store the mins and maxes for each variable for each NWP, and then the software could automatically re-scale, say, temperature data to 8-bit integers.)

And where to store the cache? Maybe start with caching for a single user, on that user's machine. Then consider a cloud caching service of some sort. For example, if lots of people request "churro-shaped" data arrays then it will be far faster to load those from a "churro-shaped" dataset cached in cloud object storage). 

