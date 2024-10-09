---
layout: post
title: "How to Decrypt HTTPS Traffic Using Wireshark: Uncovering Secure Data"
date: 2024-10-09
categories: ["wireshark","forensic"]
---

### Introduction

HTTPS has become the standard for secure communication on the web, encrypting data to protect it from prying eyes. While this encryption is crucial for privacy, there are legitimate reasons to analyze HTTPS traffic, such as diagnosing network issues, identifying potential security threats, or learning about how SSL/TLS encryption works. In this guide, we'll explore how to decrypt HTTPS traffic using Wireshark, one of the most powerful network analysis tools available. We'll break down the steps to access and analyze encrypted data, so you can gain valuable insights while maintaining ethical standards in your analysis. 

With this guide, we will using this pcap ( ) from AlexisAhmed. There is also Youtube vide walkthrough


### Understanding SSL/TLS Encryption

Data transferred over the internet or a computer network is protected by SSL (Secure Sockets Layer) encryption, as well as its more contemporary and secure counterpart, TLS (Transport Layer Security). This prohibits attackers (and Internet Service Providers) from accessing or manipulating with data sent between two nodes, generally a user's web browser and a web/app server. SSL/TLS employs both asymmetric and symmetric encryption to ensure the confidentiality and integrity of data in transit. Asymmetric encryption is used to create a secure session between a client and a server, whereas symmetric encryption is used to exchange data inside that session. 

![tls-ssl-handshake](/images/2024-10-09/tls-ssl-handshake.png)


The image above shown is a process of TLS/SSL handshake, it takes place whenever a user navigates to a website over HTTPS and the browser first begins to query the website's origin server. It also happens when API calls and DNS over HTTPS queries. 

###### TLS/SSL Handshake process:
1. TCP Handshake:
- SYN: The client initiates a TCP connection by sending SYN packet to the server
- SYN-ACK: The server acknowledges the SYN packet with a SYN-ACK
- ACK: The client responds with an ACK to complete the TCP hanshake

2. TLS/SSL Handshake:
- ClientHello: The clients sends a `ClientHello` message to the server, which includes information like supported cipher suites, TLS version, and a randomly generated number for encryption.
- ServerHello, Certificate, ServerHello: The server responds with a `ServerHello`, selecting the cipher suite and TLS version for communication. Next, it sends its digital certificate to the client for authentication. Finally, the server ends its part of the negotiation with a `ServerHelloDone` message
- ClientKeyExchange, ChangeCipherSpec, Finished: The client generates a pre-master secret, encrypts it with the server's public key, and sends it to the server. 
- ChangeCipherSec and Finished: The client and server exchange messages to confirm they are switching to the new encryption settings, completing the handshake.

