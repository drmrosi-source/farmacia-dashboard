# Project

Name: farmacia-dashboard
ID: farmacia-dashboard

## Purpose
Internal web dashboard for browsing pharmacy circulars, filtering and searching them, opening a detailed view, and downloading the related attachment when available.

## Users
Pharmacy staff using the dashboard to review incoming circulars and their required actions.

## Business objective
Make circulars faster to read and act on by presenting classification, deadlines, AI summary, key points and full text in a compact interface.

## Main technologies
- Static HTML/CSS/JavaScript in `index.html`
- Docker packaging via `Dockerfile`
- No package manager or build step

## External services
The browser reads circular data and attachment downloads from the configured Railway webhook endpoints in `index.html`.
