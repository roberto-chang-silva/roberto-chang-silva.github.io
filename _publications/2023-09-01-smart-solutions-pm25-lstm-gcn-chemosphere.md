---
title: "Smart solutions for urban health risk assessment: A PM2.5 monitoring system incorporating spatiotemporal long-short term graph convolutional network"
collection: publications
category: manuscripts
permalink: /publication/2023-09-01-smart-solutions-pm25-lstm-gcn-chemosphere
excerpt: 'This paper develops LSTGraphNet, a spatiotemporal long-short term graph convolutional network for multi-time-scale PM2.5 forecasting across a 30-station monitoring network in Gyeonggi-do, South Korea, supporting urban health risk assessment.'
date: 2023-09-01
venue: 'Chemosphere'
# slidesurl: 'https://academicpages.github.io/files/slides-chemosphere-lstgcn.pdf'
# paperurl: 'https://academicpages.github.io/files/paper-chemosphere-lstgcn.pdf'
# bibtexurl: 'https://academicpages.github.io/files/bibtex-chemosphere-lstgcn.bib'
citation: 'Chang-Silva, R., Tariq, S., Loy-Benitez, J., &amp; Yoo, C. (2023). &quot;Smart solutions for urban health risk assessment: A PM2.5 monitoring system incorporating spatiotemporal long-short term graph convolutional network.&quot; <i>Chemosphere</i>. 335, 139071. https://doi.org/10.1016/j.chemosphere.2023.139071'
---

## Abstract
Urban PM2.5 pollution poses well-documented risks to human health, yet forecasting concentrations across a monitoring network is complicated by the fact that pollutant levels are shaped by both spatial correlations among stations and complex temporal dynamics. This study proposes LSTGraphNet, a spatiotemporal long-short term graph convolutional network designed to provide reliable and fast multi-time-scale PM2.5 forecasting within a region. PM2.5 was selected as the target pollutant to reflect outdoor air quality, with a dataset collected from 30 monitoring stations across Gyeonggi-do province, South Korea (AirKorea). Using a one-week lookback data window across three temporal forecasting horizons, LSTGraphNet was benchmarked against seven AI baselines consisting of temporal air quality forecasting architectures previously tested in the air quality early warning system (EWS) domain. The proposed architecture jointly captures the spatial dependency structure of the sensor network and the temporal evolution of PM2.5 concentrations, addressing the limitations of models that treat spatial and temporal patterns independently. Results demonstrate improved predictive performance relative to conventional deep learning baselines, including standard and bi-directional LSTM architectures, the latter of which showed comparatively worse performance for PM2.5 prediction. The resulting forecasts support more accurate urban health risk assessment, offering a practical tool for early warning and targeted mitigation of PM2.5 exposure in densely populated areas.

## Key Contributions
* Proposed LSTGraphNet, a spatiotemporal long-short term graph convolutional network for network-wide, multi-time-scale PM2.5 forecasting.
* Jointly modeled spatial correlation among 30 monitoring stations in Gyeonggi-do province and temporal dependencies in pollutant concentration.
* Benchmarked the framework against seven AI baselines, including standard and bi-directional LSTM models, demonstrating improved predictive accuracy.
* Linked improved PM2.5 forecasting directly to urban public health risk assessment and early warning system applications.

<!-- ## Download & Resources
* [Download Full Paper PDF](https://academicpages.github.io/files/paper-chemosphere-lstgcn.pdf)
* [View Presentation Slides](https://academicpages.github.io/files/slides-chemosphere-lstgcn.pdf)
* [GitHub Repository for Simulation Code](https://github.com/yourusername/pm25-lst-gcn) -->
