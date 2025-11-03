# Delta Run Delta: Quick Guide

- Call `/SKN/F_SO_ST_GET_DELTA` to compare two finished runs of service-order data.
- Supply `sw_client` plus either both `im_run_no_source` and `im_run_no_target`, or a single run with `im_si_id`/`im_si_ver` so the function can resolve the pair.
- Runs must share the same service ID/version; mismatches end the call with an error message in `ex_messages`.
- On success the function returns target-run metadata, new or changed records in `so_delta_target`, deletions in `so_delta_source`, and detailed status messages in `ex_messages`.
