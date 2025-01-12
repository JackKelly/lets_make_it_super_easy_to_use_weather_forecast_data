My current obsession is to help accelerate progress in energy forecasting by making it as easy as possible to train, run, and research energy forecasting systems.

# Why?
Better energy forecasts should help reduce energy costs and CO₂ emissions.

There are a huge number of very talented energy forecasters out there. But I believe they are not able to be as effective as they could be. It's not their fault. The problem is a lack of infrastructure. 

## How do we accelerate progress in energy forecasting?
Arguably, the current craze for all-things-AI was triggered by two innovations:

First, in 2010, the ImageNet dataset evolved into the yearly [ImageNet Challenge](https://en.wikipedia.org/wiki/ImageNet#ImageNet_Challenge), where researchers competed to build the best image classification algorithm. Crucially, the ImageNet dataset was large enough to train large machine learning models but it was also easy to use: anyone with a single large hard disk could download the entire dataset.

This ease-of-use enabled a PhD student (Alex Krizhevsky) to train a convolutional neural network ([AlexNet](https://en.wikipedia.org/wiki/AlexNet)) in his bedroom on two consumer-grade GPUs. AlexNet completely dominated the 2012 ImageNet competition. This triggered renewed enthusiasm for neural networks which, ultimately, led to modern Large Language Models like ChatGPT.

In the following years, ImageNet continued to play an essential role. It provided a public "leaderboard" which clearly showed which ML algorithms performed best on the ImageNet dataset. Researchers could rigorously test which ideas worked, and which didn't.

ImageNet enabled a potent mix of friendly competition and open publishing which drove the community to quickly develop better and better image classification algorithms.

## It's hard to tell if energy forecasting is evolving
The rapid evolution we've seen in computer vision simply isn't very apparent in the world of energy forecasting. There's little sense of a year-on-year improvement in energy forecasting. This is - frankly - a terrible state to be in, and it needs to change. We're wasting money, carbon, and time.

This isn't anyone's fault. It's a systemic failure.

What's holding us back? There are, in my opinion, three main technical problems: 
1. One of the main inputs to energy forecasts is numerical weather predictions (NWPs). These datasets can be _huge_ (petabytes per year) and so are very time consuming and expensive to work with. 
2. We don't have a public, common validation dataset (like ImageNet) so we can't compare energy forecasting approaches. (Crucially, this dataset has to be large enough to enable researchers to approach the performance that for-profit companies currently obtain. More on that point in a moment.)
3. High-quality energy data is hard to obtain.

Other folks are working on fixing problem #3 (e.g. [Weave](https://weave.energy/) and the Centre for Net Zero's [OpenSynth](https://www.centrefornetzero.org/impact/open-synth)).

I feel very strongly about helping to fix problems #1 and #2.

What does that entail? I'll sketch out a roadmap. But I can't claim this is the "perfect" roadmap! So please think of this as the start of a conversation rather than a blueprint. Please comment (in the [GitHub discussion forum for this repo](https://github.com/JackKelly/lets_make_it_super_easy_to_use_weather_forecast_data/discussions))! One blindspot is that I haven't kept entirely up-to-date with academic research on energy forecasting over the last few years, so things might have happened in academia over the last few years that I've missed.

## Small datasets are not sufficient

Before we get to the road map, it's essential to make the point that tiny "toy" datasets are not sufficient. Energy forecasting companies use on the order of 1 terabyte of weather data per day (see [Dexter Energy's website](https://dexterenergy.ai)). Academics, startups, and data science teams within larger organisations need to have easy access to similar sized datasets, without having to spend years on data engineering and tens of millions of dollars on data storage.

# Use cases and benefits
- A student with a little knowledge of Python and machine learning should be able to build a state-of-the-art energy forecast in an afternoon. 
- An ambitious student should be able to implement a novel energy forecasting algorithm as part of a short project, and quickly see how well it performs against the current state of the art.
- Radically changing the shape of the data inputs to an energy forecasting algorithm should take a few hours of coding, not a few months of data preparation. 
- We should have a public leaderboard of energy forecasting algorithms, trained and validated on datasets of the size used in industry.
- Data science teams within existing energy companies (grid operators, battery optimisers, etc.) should be able to easily see which combination of data inputs and ML algorithms performs best for their situation. And then these small teams should be able to run state-of-the-art energy forecasts themselves!

# Roadmap
Lots of people are already doing great work! For example: the [Pangeo community](https://www.pangeo.io); the [`xarray`](https://docs.xarray.dev/en/stable) and [`kerchunk`](https://fsspec.github.io/kerchunk) Python packages; NWP providers including NOAA, ECMWF, the UK Met Office; and many others! Below is my proposal for what I'm planning to push on over the next few years... This will hopefully integrate nicely with existing tools like `xarray`, and build on the great work of the NWP community. OK, here's my proposal...

1. Make it trivial to lazily open petabyte-sized numerical weather datasets. To this end, I'm currently working on [`hypergrib`](https://github.com/jackkelly/hypergrib) and [`explore_nwps`](https://github.com/JackKelly/explore_nwps). See [this blog post](https://openclimatefix.org/post/lazy-loading-making-it-easier-to-access-vast-datasets-of-weather-satellite-data) for more details.
2. But [some read patterns will never be well-served by reading directly from GRIB](https://github.com/jackkelly/hypergrib?tab=readme-ov-file#which-read-patterns-will-perform-well-with-hypergrib). This is because each GRIB message extends across the entire horizontal geospatial extent of the dataset so reading data for a small number of geospatial locations from GRIB will always be inefficient. So the next project I plan to work on will be an automatic caching system. Again, the focus will be on making life as easy as possible for the developer. See below for more details.

Once these essential components are in place, some other cool projects might include (in no particular order):
- Implement an open-source, proof-of-concept, state-of-the-art ML energy forecasting algorithm. Crucially, it should be a very small codebase (a few hundred lines of Python?) because `hypergrib` and other tools should make it trivial to get NWP data from multiple providers into the correct shape for ML training.
- Build a public leaderboard of different energy forecasting algorithms, measured against a standard validation dataset (using the types of huge datasets that are used in industry, rather than toy academic datasets). Anyone could contribute algorithms to the leaderboard.
- An analysis tool (with a UI a bit like [Windy.com](https://windy.com)'s UI?) for comparing different NWPs against each other and against ground truth. 
- On the fly processing and analytics. E.g. reprojection.
- Distribute `hypergrib`'s workload across multiple machines. So, for example, users can get acceptable IO performance even if they ask for "churro-shaped" data arrays.

# Details

## Caching GRIB data so you still get high performance for read patterns which don't fit with GRIB's data layout
What if users want to read long timeseries for a small number of geographical points (a "churro-shaped" array)? No amount of `hypergrib` trickery will get round the physical constraint that each GRIB message is a 2D array representing a horizontal plane.

So we'll need a way to cache these "churro-shaped" arrays.

Perhaps the "dream" would be a completely transparent caching layer. The user would just say "I want data in this shape; and my read pattern will look like this" and the software transparently creates a high-speed cache of that data (in cloud object storage or local disks). Perhaps using the `Icechunk` storage engine or `zarrs`. It must also be very easy to update the cached dataset with new NWP data (e.g. so the same cached dataset can be used for ML training and for inference.).

One idea would be to write a Rust command-line app which creates Zarrs from `hypergrib` data. e.g. "Get data over the United Kingdom from 2017 to today from these three NWPs, and save to a Zarr with this chunk shape, and distribute the workload across 8 VMs". And, later, make it easy to update the Zarr with recent data.

But what if users want to perform arbitrary processing on the data as it's being moved from GRIB to Zarr? Can we optimise & orchestrate exposing the GRIB data to Python, running Python processing, and save to Zarr? Perhaps, at least to start with, we say that it's not possible to perform any computation as the data is being copied to a local cache (other than lossless compression, including reducing the numerical precision. For example, we could store the mins and maxes for each variable for each NWP, and then the software could automatically re-scale, say, temperature data to 8-bit integers.)

And where to store the cache? Maybe start with caching for a single user, on that user's machine. Then consider a cloud caching service of some sort. For example, if lots of people request "churro-shaped" data arrays then it will be far faster to load those from a "churro-shaped" dataset cached in cloud object storage). 

## Tool for comparing NWPs against each other, and against ground truth
Where would `hypergrib` run? Perhaps _in_ the browser, using `wasm`?! (but [tokio's `rt-multi-thread` feature doesn't work on `wasm`](https://docs.rs/tokio_wasi/latest/tokio/#wasm-support), which might be a deal-breaker.) Or perhaps run a web service in the cloud, close to the data, across multiple machines. And [expose a standards compliant API like Environmental Data Retrieval](https://github.com/JackKelly/hypergrib/issues/19) for the front-end?

Perhaps include tools to look at correlation between NWPs and actual measurements, including actual measurements from the energy system (like wind power production or solar power generation).

# Related projects

I can't take any credit for these projects but these projects all help towards the aim of making it super-easy for folks to run energy forecasts:

(alphabetical order)

- [dynamical.org](https://dynamical.org)
- [Pangeo](https://www.pangeo.io)
- [Weave](https://weave.energy)

# Further info
Please see [this blog post for more info on this idea](https://openclimatefix.org/post/lazy-loading-making-it-easier-to-access-vast-datasets-of-weather-satellite-data).
