---
title: "Graph Attention Sensor Transformer for Industrial Emission Forecasting: A Comparative Study Against Classical and Deep Learning Baselines"
collection: publications
category: manuscripts
permalink: /publication/2026-01-15-graph-attention-sensor-transformer-emissions
excerpt: 'This paper introduces a Graph Attention Sensor Transformer that models inter-sensor spatial dependencies and temporal dynamics for industrial emission forecasting, benchmarked against classical statistical and deep learning baselines.'
date: 2026-01-15
venue: 'Mathematical Geosciences'
# slidesurl: 'https://academicpages.github.io/files/slides-mathgeo-gast-emissions.pdf'
# paperurl: 'https://academicpages.github.io/files/paper-mathgeo-gast-emissions.pdf'
# bibtexurl: 'https://academicpages.github.io/files/bibtex-mathgeo-gast-emissions.bib'
citation: 'Chang-Silva, R., Song, N., Lee, K., &amp; Park, S. (2026). &quot;Graph Attention Sensor Transformer for Industrial Emission Forecasting: A Comparative Study Against Classical and Deep Learning Baselines.&quot; <i>Mathematical Geosciences</i>. https://doi.org/10.1007/s11004-026-10320-x'
---

## Abstract
Industrial emission monitoring networks generate dense, multivariate sensor streams whose forecasting is complicated by strong spatial coupling among sensors and by long-range, non-linear temporal dependencies in the underlying processes. This paper introduces a Graph Attention Sensor Transformer (GAST) that represents an industrial sensor network as a graph and applies attention-based message passing to learn dynamic inter-sensor dependencies, combined with a transformer-based temporal encoder to capture long-range dependencies in emission time series. Unlike static-graph approaches, the attention mechanism adaptively re-weights the influence of neighboring sensors as operating conditions change, allowing the model to respond to shifts such as process transitions or equipment load changes. The proposed architecture is evaluated on real industrial emission monitoring data and benchmarked comprehensively against classical statistical forecasting methods (e.g., ARIMA-family models) and established deep learning baselines (including LSTM, GRU, and standard graph convolutional networks). Results show that GAST achieves consistent improvements in forecasting accuracy across multiple horizons, particularly during periods of rapid emission fluctuation, while providing interpretable attention weights that reveal which sensors most strongly influence a given prediction. The findings support the use of graph-attention transformer architectures as a practical tool for proactive emissions management and regulatory compliance monitoring in industrial settings.

## Key Contributions
* Introduced the Graph Attention Sensor Transformer (GAST), coupling dynamic graph attention with a transformer temporal encoder for multivariate industrial emission forecasting.
* Conducted a systematic comparative evaluation against classical statistical models and deep learning baselines (LSTM, GRU, GCN) on real industrial sensor network data.
* Demonstrated improved forecasting accuracy during periods of rapid emission fluctuation across multiple prediction horizons.
* Provided interpretable attention weights identifying the sensors most influential to each forecast, supporting emissions management and compliance monitoring.

<!-- ## Download & Resources
* [Download Full Paper PDF](https://academicpages.github.io/files/paper-mathgeo-gast-emissions.pdf)
* [View Presentation Slides](https://academicpages.github.io/files/slides-mathgeo-gast-emissions.pdf)
* [GitHub Repository for Simulation Code](https://github.com/yourusername/graph-attention-sensor-transformer) -->
