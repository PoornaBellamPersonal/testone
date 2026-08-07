# POC #5.2 changes

- Added 300-name near-real insurance/commission attribute catalog.
- Added `nearreal-smoke`, `nearreal-medium`, and `nearreal-target` seed profiles.
- Target profile: 300K sparse rules / 3M transactions.
- Target generator produces 60K logical programs and ~3.687M rule-condition rows (~12.29 conditions/rule).
- All 300 registered attributes are referenced by the generated rule population; any individual rule uses only a small subset.
- Added realistic carrier/product/channel/producer/policy/compensation values.
- Added ~3% deliberate no-match transactions and moderate traffic skew.
- Added configurable extra populated transaction attributes (`rfp.demo.tx-extra-attributes`).
- Replaced bitmap-per-distinct-value equality indexes with compact posting lists to support high-cardinality attributes safely.
- Reworked rule loading to stream ordered rule/condition rows instead of building a large rule-to-conditions HashMap.
- Increased transaction payload column capacity for wider dynamic records.
- Added large-heap run and target-seed BAT files.
- Added optional `java21-env.bat` hook to scripts.
- Added full 300-attribute CSV catalog and near-real rule examples.
