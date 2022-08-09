---
layout: post
title: "Send amf package using python"
tags: [python,amf]
---

```
#!/usr/bin/env python
#coding:utf8
import pyamf
import requests
import urllib
from pyamf import remoting
from BaseHTTPServer import BaseHTTPRequestHandler, HTTPServer
class MyHTTPHandler(BaseHTTPRequestHandler):
    def getValue(self, key):
        arr=self.path.split("?")
        if len(arr)<2:
            return ""
        arr=arr[1].split("&")
        for i in range(0,len(arr)):
            tmp=arr[i].split("=")
            if tmp[0]==key:
                return tmp[1] if tmp[1] else ""
        return ""
    def do_GET(self):
        data=[urllib.unquote(self.getValue("email")),urllib.unquote(self.getValue("password"))]
        req = remoting.Request('method', body=(data))
        env = remoting.Envelope(pyamf.AMF0)
        env.bodies = [('/1',req)]
        data = bytes(remoting.encode(env).read())
        url = 'url' 
        try:
            req = requests.post(url=url,data=data,headers={'Content-Type':'application/x-amf'})
            resp = remoting.decode(req.content)
            html = resp.bodies[0][1].body
            self.send_response(200)
            self.send_header("Content-type","text/html")
            self.end_headers()
            self.wfile.write(html)
        except Exception as e:
            print str(e)
server = HTTPServer(("", 8000), MyHTTPHandler)
server.serve_forever()
```
