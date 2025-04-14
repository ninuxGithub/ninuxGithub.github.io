---
title: Shell query
author: ninuxGithub
layout: post
date: 2025-4-14 14:50:13
description: "Shell Usage"
tag: js
---

### Shell


```shell

while true; do
    curl --location 'http://ghr.derbysoft-test.com/ghr/ghr.ci' \
    --header 'Content-Type: application/json' \
    --data '{
        "method": "getContextHotels",
        "params": [
            "ACCOR",
            [
                "1434"
            ],
            [
                "Hotel.Context.ACCOR.ID",
                "Name.EN-US",
                "Hotel.Context.ACCOR.ChainCode",
                "Hotel.Context.ACCOR.BrandCode",
                "CurrencyCode",
                "ContactInfo.1.Address.1.CityName.EN-US",
                "Location.CountryCode",
                "ContactInfo.1.Address.1.CountryName.EN-US",
                "Description.EN-US",
                "TimeZone.Code",
                "PolicyInfo.CheckInTime",
                "PolicyInfo.CheckOutTime",
                "PolicyInfo.UsualStayFreeCutOffAge",
                "PolicyInfo.KidsStayFree",
                "Location.Longitude",
                "Location.Latitude"
            ]
        ]
    }'
sleep 3
```