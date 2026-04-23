# Astronomy workloads

The homelab hosts astronomy and data-analysis workloads: large catalog
queries, Dask clusters, and telescope-adjacent tooling.

## What is here

- Which tools run where (LSDB, lightkurve, yt, sky-explorer tooling).
- How Dask/Ray clusters are deployed and scaled on Kubernetes.
- Storage and data locality notes for catalog-scale queries.

## Upstream contributions

Work that came out of running this lab:

- [lsdb#1325](https://github.com/astronomy-commons/lsdb/pull/1325)
- [lightkurve#1553](https://github.com/lightkurve/lightkurve/pull/1553)
- [yt#4391](https://github.com/yt-project/yt/pull/4391)

TODO expand with dataset sizes (orders of magnitude, not exact numbers)
and typical query patterns.
