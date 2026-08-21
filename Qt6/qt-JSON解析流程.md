JSON解析：

```json
{
  "location": {
    "name": "Shang",
    "region": "Paktia",
    "country": "Afghanistan",
    "lat": 33.385,
    "lon": 69.3872,
    "tz_id": "Asia/Kabul",
    "localtime_epoch": 1783671995,
    "localtime": "2026-07-10 12:56"
  },
  "current": {
    "last_updated_epoch": 1783671300,
    "last_updated": "2026-07-10 12:45",
    "temp_c": 27.9,
    "temp_f": 82.1,
    "is_day": 1,
    "condition": {
      "text": "Sunny",
      "icon": "//cdn.weatherapi.com/weather/64x64/day/113.png",
      "code": 1000
    },
    "wind_mph": 9.8,
    "wind_kph": 15.8,
    "wind_degree": 315,
    "wind_dir": "NW",
    "pressure_mb": 1004,
    "pressure_in": 29.65,
    "precip_mm": 0,
    "precip_in": 0,
    "humidity": 10,
    "cloud": 0,
    "feelslike_c": 20.8,
    "feelslike_f": 69.4,
    "windchill_c": 27.9,
    "windchill_f": 82.1,
    "heatindex_c": 25.9,
    "heatindex_f": 78.6,
    "dewpoint_c": -6.9,
    "dewpoint_f": 19.5,
    "vis_km": 10,
    "vis_miles": 6,
    "uv": 12.9,
    "gust_mph": 11.3,
    "gust_kph": 18.2,
    "will_it_rain": 0,
    "chance_of_rain": 0,
    "will_it_snow": 0,
    "chance_of_snow": 0,
    "short_rad": 1029.12,
    "diff_rad": 148.34,
    "dni": 0,
    "gti": 0
  }
}
```

```c++
// qt6 解析JSON
void BackendHelper::handleRequestFinished(QNetworkReply *reply)
{
    //handle when the network manager fired the signal
    if(reply->error() == QNetworkReply::NoError) {

        QByteArray replyBytes = reply->readAll();
        qDebug().noquote() << replyBytes;

        QJsonParseError parseError;
        QJsonDocument doc = QJsonDocument::fromJson(replyBytes, &parseError);
        //check for error in parsing
        if(parseError.error == QJsonParseError::NoError) {
            //json is okay
            QJsonObject entireJson = doc.object();
            QJsonObject locationData = entireJson.value("location").toObject();
            QJsonObject currentData = entireJson.value("current").toObject();

            QString name = locationData.value("name").toString();
            QString country = locationData.value("country").toString();
            double tempC = currentData.value("temp_c").toDouble();

            QVariantMap result;
            result["name"] = name;
            result["country"] = country;
            result["temp_celcius"] = tempC;
            emit locationDataReceived(result);
        } else {
            qDebug() << "Some error in parsing " << parseError.errorString();
        }


    } else {
        qDebug() << "Some error occurred " << reply->errorString();
    }
    // very important
    reply->deleteLater();
}

```

