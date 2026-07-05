# Datasets

Three datasets span the e-mail, SMS, and voice social-engineering channels. All are mapped to
the same five-construct schema (see [methodology.md](methodology.md)) so that cross-channel score
differences reflect reasoning skill, not schema mismatch. All records are anonymised; no model
was asked to generate live phishing content — prompts referenced behavioural constructs only.

## Overview

| Channel | Raw rows | Cleaned rows | Malicious % | Features |
|---------|----------|--------------|-------------|----------|
| E-mail phishing | 88,647 | 59,788 | 31.15% | 10 |
| SMS smishing | 67,008 | 67,008 | 39.07% | 10 |
| Synthetic vishing | 60,000 | 60,000 | 30.00% | 10 |

The collection strategy follows Opara, Chen and Wei (2024).

## 1. E-mail phishing — primary benchmark

- **Source:** open e-mail phishing corpus of Vrbančič, Fister and Podgorelec (2020).
- **Role:** primary benchmark for DAG construction and LLM alignment scoring.
- **Cleaning:** 88,647 raw rows reduced to 59,788 after deduplication/normalisation.
- **Composition:** 31.15% malicious; 10 engineered features per record.
- **Feature character:** URL- and domain-derived indicators (shortened URLs, redirect and
  resolved-IP counts, index presence, TLS/SSL status, embedded email-in-URL, domain age,
  response time).

## 2. SMS smishing — evasion-optimised

- **Source:** smishing corpus released by Salman, Ikram and Kaafar (2024), studying evasive SMS
  techniques.
- **Role:** cross-channel robustness (text channel).
- **Composition:** 67,008 rows; 39.07% malicious; 10 features.
- **Feature character:** engineered rows derived from a two-field raw spam–ham archive — brand
  references, mimicry, and text-related signals.

## 3. Synthetic vishing — CVAE-generated

- **Source:** synthetic voice-phishing simulation generated with a Conditional Variational
  Auto-Encoder (CVAE); contains **no voice recordings**.
- **Role:** cross-domain generalisation without privacy risk.
- **Composition:** 60,000 rows; 30.00% malicious; 10 features.
- **Feature character:** synthetic binary flags for call scenarios — caller-ID spoofing,
  authority-based scripts, and related cues.
- **Why CVAE:** it preserves the latent causal scaffold while allowing selective removal of
  surface cues, so cross-channel shifts can be attributed to the data rather than the discovery
  engine.

## Feature-to-construct mapping

Each channel loads its own YAML map (`phishing.yaml`, `smishing.yaml`, `vishing.yaml`) assigning
technical features to the five behavioural constructs:

| Construct | Representative technical features |
|-----------|-----------------------------------|
| Obfuscation | Shortened URLs, redirect count, resolved-IP count |
| Trust | URL in Google index, TLS/SSL certified, domain in Google index |
| Authority | Email address embedded within URL |
| Deception | Time of domain activation (domain age) |
| Urgency | Website response time |

Explicit feature→construct mapping (cf. Guo et al., 2021) ensures the three channels are
compared on a common behavioural basis rather than channel-specific surface artefacts.
