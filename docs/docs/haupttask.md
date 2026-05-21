# Opaque Gruppen

Diese Implementierung wurde spezifiziert in QEP https://github.com/qgis/QGIS-Enhancement-Proposals/blob/master/qep-402-opaque-layergroups.md

In den WMS-Eigenschaften gibt es die neue Option, den Abfragemodus für Gruppen auf "opaque" zu setzen:

<img width="937" height="527" alt="Screenshot From 2026-05-15 10-53-00" src="https://github.com/user-attachments/assets/72293f15-c330-43b4-a726-3d820c36931f" />

Wenn eine Gruppe opaque ist, bedeutet dies, dass die folgenden Abfragen die Kind-Elemente der opaque Gruppe nicht auflisten (das heisst, sie listen "leere" Gruppen auf):
- GetCapabilities
- GetProjectSettings
  - hat zusätzlich 'opaque' in den Attributen einer opaque Gruppe
  - listet zusätzlich die Kind-Layer einer opaque Gruppe nicht in der `LayerDrawingOrder` auf
- GetContext
  - listet die Kind-Layer einer opaque Gruppe nicht auf (ebenso wenig die opaque Gruppen, da Gruppen generell nicht aufgelistet werden)

Grundsätzlich verhalten sich Abfragen auf opaque Layergruppen genauso wie Abfragen auf normalen Gruppen. Wenn jedoch ein Kind-Element einer opaque Gruppe abgefragt wird, wird ein `LayerNotDefined`-Fehler zurückgegeben. Dies gilt für:
- GetMap
  - application/dxf
  - application/pdf
  - normal
- GetFeatureInfo
- GetLegendGraphic / GetLegendGraphics
- GetStyle / GetStyles
- GetPrint
- DescribeLayer 

Der Grund dafür ist, dass die Kind-Elemente einer opaque Gruppe ausgeblendet sein sollten. Dennoch werden bei der Abfrage einer opaque Gruppe alle Kind-Elemente gerendert (bei GetMap, GetPrint) oder aufgelistet (bei GetFeatureInfo / GetLegendGraphics / GetStyles).

Keine Änderungen bei:
- GetSchemaExtension

<div class="page-break"></div>

## Beispiel für `GetCapabilities`

<img width="341" height="125" alt="image" src="https://github.com/user-attachments/assets/52d49560-d5b1-4e04-a7d9-d197f76e1701" />

`road-group` ist opaque und `street-group` ist es nicht.


```xml
<BoundingBox CRS="EPSG:4326" maxx="47.528592" maxy="8.772493" minx="47.505523" miny="8.756412"/>
<Layer queryable="1">
  <Name>street-group</Name>
  <Title>street-group</Title>
  <CRS>CRS:84</CRS>
  <CRS>EPSG:4326</CRS>
  <CRS>EPSG:3857</CRS>### T
  [...]
  <Layer queryable="1">
    <Name>streetline</Name>
    <Title>streetline</Title>
    <CRS>CRS:84</CRS>
    <CRS>EPSG:4326</CRS>
    <CRS>EPSG:3857</CRS>
    [...]
  </Layer>
  <Layer queryable="1">
    <Name>streetarea</Name>
    <Title>streetarea</Title>
    <CRS>CRS:84</CRS>
    <CRS>EPSG:4326</CRS>
    <CRS>EPSG:3857</CRS>
    [...]
  </Layer>
</Layer>
<Layer queryable="1">
  <Name>road-group</Name>
  <Title>road-group</Title>
  <CRS>CRS:84</CRS>
  <CRS>EPSG:4326</CRS>
  <CRS>EPSG:3857</CRS>
  [...]
</Layer>
```

Oder im QGIS Datenquellenmanager:

<img width="427" height="150" alt="image" src="https://github.com/user-attachments/assets/580c9a6b-aac3-4d13-a4a6-f77949d4b44c" />

<div class="page-break"></div>

## Technische Info

Bei allgemeinen Abfragen wie `GetCapabilities`, `GetProjectInfo` etc. listen wir die Kind-Elemente der Gruppe einfach nicht auf, wenn der Abfragemodus "opaque" ist. Nur bei der Layer-Zeichenreihenfolge (LayerDrawingOrder) in `GetCapabilities` müssen wir prüfen, welche Layer "sichtbar" sind und welche nicht. Dies tun wir mithilfe der Hilfsmethode `collectAcceptableLayersAndRequestNames`.

In `collectAcceptableLayersAndRequestNames` gehen wir den Layer-Baum des Projekts durch und schreiben alle Layer, die gerendert wer Gruppen

Diese Impden dürfen, in ein `QHash` zusammen mit der Information, wie sie aktuell requested werden (für den Fall, dass wir eine Liste von `requestedLayerNames` übergeben). Das bedeutet, dass ohne die Übergabe einer Liste von `requestedLayerNames` - wie wir es in `GetCapabilities` tun - alle Layer aufgelistet werden, die sich nicht in einer opaque Gruppe befinden. Andernfalls werden alle Layer aufgelistet, die requested wurden (explizit über den Namen oder über ihre Gruppe).

Bei Abfragen wie `GetFeatureInfo`, `GetMap` etc. (implementiert in `QgsWmsRenderContext`) und `DescribeLayer` (implementiert in `writeDescribeLayer`) prüfen wir vor dem Rendern (oder Beschreiben), ob der Layer abgefragt (oder beschrieben) werden darf. Zudem geben wir einen `LayerNotDefined`-Fehler aus, wenn ein LAYER über Parameter requested wird, die nicht zu einem abfragbaren Layer passen (z. B. weil er nicht existiert oder in einer opaque Gruppe versteckt ist).

Wenn gleichnamige Layer vorhanden sind, ist das Verhalten wie folgt:
- Ein Layer in einer opaque Gruppe wird nicht mit einem anderen gleichnamigen Layer "zusammengeführt" (merged), wenn die opaque Gruppe requested wird.
- Ein Layer wird nicht mit einem anderen gleichnamigen Layer in einer opaque Gruppe "zusammengeführt", wenn der Layer requested wird.

## Related issues

Es gibt unerwünschtes Verhalten, dessen Behebung diese Lösung verbessern würde.

Da es diese Anforderung jedoch nicht direkt betrifft, sind sie nicht Teil der Implementation.

- Wenn ein Layer excluded ist, wird bei der Abfrage der Gruppe, die den Layer enthält, ein Fehler zurückgegeben (anstatt einfach alles andere zurückzugeben): https://github.com/qgis/QGIS/issues/65801
- Eine GetFeatureInfo-Abfrage auf gleichnamigen Layern führt die Features nicht "zusammen": https://github.com/qgis/QGIS/issues/65989 Dies funktioniert bei GetMap etc.
- Ein Layer würde sich nicht "zusammenführen" mit einer gleichnamigen Gruppe (oder opaque Gruppe) und deren Layern, wenn der Layer requested wird (was er tun sollte), und eine Gruppe (oder opaque Gruppe) und deren Layer würden sich nicht "zusammenführen" mit einem gleichnamigen Layer, wenn die Gruppe (oder opaque Gruppe) requested wird (was sie tun sollte). Es wird nur der Layer gerendert (und nicht die Gruppe): https://github.com/qgis/QGIS/issues/66134

<div class="page-break"></div>

# GetFeatureInfo JSON Layer + Gruppen

## Layer Name

Diese Implementierung wurde spezifiziert in QEP https://github.com/qgis/QGIS-Enhancement-Proposals/blob/master/qep-410-json-layer-name.md

Eine WMS-`GetFeatureInfo`-Anfrage gibt ein JSON zurück, das den Layernamen (oder den Short-Name beziehungsweise die Layer-ID, je nach Einstellung) als **featureType** enthält:

```json
{
  "features": [
    {
      "featureType": "streetline",
      "geometry": null,
      "id": "streetline.2",
      "properties": {
        "id": 2,
        "name": "Albert-Einstein-Strasse"
      },
      "type": "Feature"
    },
    {
      "featureType": "streetline",
      "geometry": null,
      "id": "streetline.3",
      "properties": {
        "id": 3,
        "name": "Schoorenstrasse"
      },
      "type": "Feature"
    }
  ],
  "type": "FeatureCollection"
}
```

Das Element `featureType` wird in der "OGC Features and Geometries JSON" (JSON-FG)-Spezifikation eingeführt (https://docs.ogc.org/is/21-045r1/21-045r1.html#feature-types). In GeoJSON ist dieses zusätzliche Element als Foreign Member zulässig.

## Gruppen Name

Bei einer WMS-`GetFeatureInfo`-Anfrage geben wir zusätzlich zum Layername im `featureType` auch den angeforderten WMS-Namen als `qgis:requestedWmsName` zurück, sofern der angeforderte Name nicht dem Layernamen selbst entspricht (was bedeutet, dass es sich um die Gruppe handeln muss).

Das heisst, wenn der Layer abgefragt hat wird, kriegen wir nach wie vor nur den `featureType`:

```json
  "features": [
    {
      "featureType": "streetline",
      "geometry": null,
      "id": "streetline.2",
      "properties": {
        "id": 2,
        "name": "Albert-Einstein-Strasse"
      },
      "type": "Feature"
    }
  ]
```

Wenn aber die übergeordnete Gruppe (opaque oder nicht) angefragt wird, haben wir zusätzlich der Name der Gruppe drin:

```json
  "features": [
    {
      "featureType": "streetline",
      "geometry": null,
      "id": "streetline.2",
      "properties": {
        "id": 2,
        "name": "Albert-Einstein-Strasse"
      },
      "qgis:requestedWmsName": "street-group",
      "type": "Feature"
    }
  ]
```

Wir geben immer das oberste übergeordnete Element (höchster Vorfahre) zurück. Das bedeutet für eine Layer-Struktur wie:

```
- street-group
  |- big-street-group
     |- groups-named-after-scientists
        |- streetline
```

Bei `QUERY_LAYERS=streetline` -> kein `qgis:requestedWmsName`
Bei `QUERY_LAYERS=streetline,big-street-group` -> `qgis:requestedWmsName` ist "big-street-group"
Bei `QUERY_LAYERS=streetline,big-street-group, street-group` -> `qgis:requestedWmsName` ist "street-group"
Bei `QUERY_LAYERS=big-street-group` -> `qgis:requestedWmsName` ist "gbig-street-grouproup2"
etc.

## Technische Info

Den Namespace `qgis:` verwenden wir für Foreign Member, die QGIS-spezifisch sind. `featureType` ist hingegen ein offizielles JSON-FG-Element und sollte kein Präfix erhalten. 
Abdeckungslayer in QGIS WMS und Layer- bzw. Gruppennamen in JSON Antworten

<div class="page-break"></div>

# Source Code

- WMS Opaque layergroups in QGIS server: https://github.com/qgis/QGIS/pull/65836
- Layer Name in WMS GetFeatureInfo JSON Result as featureType (JSON- FG) https://github.com/qgis/QGIS/pull/66008
- WMS qgis:requestedWmsName (Group) in GetFeatureInfo JSON Result as foreign member https://github.com/qgis/QGIS/pull/66166