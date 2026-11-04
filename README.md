# Customs Risk Predictive Project

Predictive risk scoring on synthetic customs clearance data (5,250 shipments, MEA region;
4,746 after cleaning): which shipments are likely to clear slowly, and which are likely to be
under-declared.

Status: in progress. Built step by step alongside a machine learning course.
Current stage: exploratory analysis complete. Next: significance tests, train/test design and models.
Progress, decisions and findings are logged in PROJECT_LOG.md.

## What is here so far
- `01_Data_Cleaning.ipynb`: cleaning of the raw file (5,250 to 4,746 rows)
- `02_Feature_Engineering.ipynb`: time features, licence flag, valuation-gap target
- `03_EDA.ipynb`: exploratory charts (saved in `images/`)
- `PROJECT_LOG.md`: dated record of steps, numbers and decisions

## Early findings (exploratory, not yet tested)
- About 26% of shipments clear slowly and 18% show a large valuation discrepancy, so accuracy would
  be a misleading measure and models are judged against these base rates.
- Slow clearance is driven mainly by port and weekend entry. Valuation discrepancy is driven by
  product type, supplier and origin, not by port.
- Weight, volume and broker fee show almost no link to slow clearance.

![Slow rate by port](images/03_slow_rate_by_port.png)