---
title: "Notes"
date: 2024-07-31T22:49:04+10:00
draft: true
tags: ['kubernetes', 'eli5']
categories: ['tech']
---



Install and run Ollama in WSL

```
netsh interface portproxy add v4tov4 listenport=11434 listenaddress=192.168.0.4 connectport=11434 connectaddress=172.27.241.132

PS C:\Windows\system32> netsh advfirewall firewall add rule name="Allowing LAN connections" dir=in action=allow protocol=TCP localport=11434   

```