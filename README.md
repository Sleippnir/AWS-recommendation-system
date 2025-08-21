# Consolidated Recommendation System Dashboard

This repository contains the source code for a single-page interactive web application that visualizes the complete architecture, cloud service mapping, and real-time data flow for a large-scale recommendation system.

## Overview

This project dashboard is a minimalist tool designed for project stakeholders. It consolidates three project documents into a single, easy-to-navigate interface:

1.  **The  Plan:** A step-by-step plan detailing the system's architecture, training protocols, and operational strategies.
2.  **AWS Service Mapping:** A direct mapping of each system component to its corresponding AWS service.
3.  **Interactive Data Lineage Graph:** A dynamic visualization of the real-time request/response cycle, showing data flow and target latencies for each stage of the serving pipeline.

The primary goal is to make a complex system architecture intuitive, explorable, and easy to understand.

## Project Goals & Motivation

The primary business objective for this recommendation system is to **increase the average order value** by providing intelligent, personalized product suggestions. The system is designed to address several key challenges and use cases:

* **Contextual Recommendations:** The system must deliver relevant recommendations across three distinct user scenarios:
    1.  General recommendations on a business's homepage.
    2.  Similar item recommendations on a product detail page.
    3.  Complementary item recommendations in the shopping cart.
* **Real-Time Accuracy:** The system must handle the dynamic nature of inventory, ensuring that products that are no longer offered or are out-of-stock are not recommended.
* **Comprehensive Cold-Start Solution:** It must provide effective recommendations for new users, new businesses, and new products from day one.
* **Fairness & Discovery:** The architecture includes mechanisms to mitigate popularity bias, preventing an over-reliance on popular chain restaurants or items and promoting the discovery of long-tail products.
* **High-Scale Performance:** The system is architected to meet demanding non-functional requirements, including serving 100,000 recommendations per second for a catalog of 10 million products and 20 million active users.

## Features

* **Tabbed Interface:** Easily switch between the three core documents without leaving the page.
* **Interactive Data Flow Graph:** Hover over components in the data lineage graph to highlight their connections and see detailed descriptions of their roles and performance targets.
* **Responsive Design:** The dashboard is fully responsive and accessible on desktop, tablet, and mobile devices.
* **Zero Dependencies:** The entire application is a single `index.html` file with no local dependencies, making it extremely portable.

## Live Demo

You can view the live dashboard hosted on GitHub Pages here:

[**Recommendation System Dashboard**](https://sleippnir.github.io/AWS-recommendation-system/)

## Tech Stack

* **HTML5**
* **Tailwind CSS** (via CDN)
* **Vanilla JavaScript** (for all interactivity and graph rendering)
