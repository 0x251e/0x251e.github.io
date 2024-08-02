---
layout: post
title:  "Wireshark For Geolocating IP Address"
date:   2018-11-04 12:32:45 +0100
categories: learning
---



# Wireshark for Geolocating IP Addresses

Wireshark is a fantastic tool for cybersecurity and network enthusiasts to perform network analysis and troubleshooting. Some of the basic features include packet capture, exporting data, analyzing bandwidth usage, etc.

For this topic, we are going to dive deeper into geolocating IP addresses and filtering by city, country, and even ASN number. Unfortunately, this feature is not available as a default tool. However, we can utilize databases as a plugin to add this functionality.

There are two databases available:
- **MaxMind GeoIP2**: Paid version with higher accuracy.
- **GeoLite2**: Free version.

## Steps to Enable GeoIP in Wireshark

1. **Download the database**
2. **Install the plugin**
3. **Configure Wireshark**

## Benefits of GeoIP

### Assist in Forensic Investigations
- Provides insights and reveals the origin of network traffic involved in security incidents and breaches.

### Geographic Visualization
- Allows visualization of network traffic on maps, providing an overview and intuitive understanding of network behavior.

### Performance Optimization
- Identifies latency issues or inefficiencies in routing configuration based on endpoints in different locations.

### Security Analysis for SOC
- Helps detect and investigate potential security threats like DDoS attacks, malware infections, or unauthorized access.

