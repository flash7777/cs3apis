# CS3 API: Batch & Streaming Erweiterungen

Status: TODO / Diskussionsvorlage
Stand: 2026-06-28
Kontext: cloudd (headless files-on-demand daemon), openvfs FUSE Integration

## Problem

Das CS3-Protokoll hat 119 RPCs, aber **keinen einzigen Batch- oder Streaming-RPC
für Daten- oder Metadata-Operationen**. Datentransfer läuft ausschließlich
out-of-band über HTTP (InitiateFileUpload/Download → URL → HTTP PUT/GET).

Die zwei existierenden Streaming-RPCs (ListContainerStream, ListRecycleStream)
waren im Gateway nicht implementiert — ListContainerStream haben wir gepatcht
(reva feature/gateway-listcontainerstream).

Für Clients die viele Dateien on-demand hydratisieren (files-on-demand, VFS)
wird der HTTP-Overhead pro Request zum Bottleneck:
- ~50ms pro Datei über TLS 1.3 + HTTP/2 (10G LAN)
- Bei 100 Dateien: 5 Sekunden sequentiell, ~2.5s mit 10x parallel
- Batch würde das auf 1 Roundtrip reduzieren

## Vorgeschlagene Erweiterungen

### 1. BatchStat (Prio: hoch)

Mehrere Dateien in einem Request statten — häufigstes Pattern bei Sync/VFS.

```protobuf
rpc BatchStat(BatchStatRequest) returns (BatchStatResponse);

message BatchStatRequest {
  repeated cs3.storage.provider.v1beta1.Reference refs = 1;
  repeated string arbitrary_metadata_keys = 2;
  google.protobuf.FieldMask field_mask = 3;
}

message BatchStatResponse {
  cs3.rpc.v1beta1.Status status = 1;
  // Key: Index in refs, Value: ResourceInfo oder Error-Status
  repeated StatResult results = 2;
}

message StatResult {
  cs3.rpc.v1beta1.Status status = 1;
  cs3.storage.provider.v1beta1.ResourceInfo info = 2;
}
```

### 2. BatchSetArbitraryMetadata (Prio: mittel)

Metadata auf mehrere Dateien in einem Request setzen.

```protobuf
rpc BatchSetArbitraryMetadata(BatchSetArbitraryMetadataRequest)
    returns (BatchSetArbitraryMetadataResponse);

message BatchSetArbitraryMetadataRequest {
  repeated SetArbitraryMetadataRequest requests = 1;
}

message BatchSetArbitraryMetadataResponse {
  repeated cs3.rpc.v1beta1.Status statuses = 1;
}
```

### 3. BatchInitiateFileDownload (Prio: mittel)

Mehrere Download-URLs in einem Request anfordern.

```protobuf
rpc BatchInitiateFileDownload(BatchInitiateFileDownloadRequest)
    returns (BatchInitiateFileDownloadResponse);

message BatchInitiateFileDownloadRequest {
  repeated InitiateFileDownloadRequest requests = 1;
}

message BatchInitiateFileDownloadResponse {
  cs3.rpc.v1beta1.Status status = 1;
  repeated InitiateFileDownloadResponse responses = 2;
}
```

### 4. ListContainerStream im Gateway (Prio: erledigt)

Bereits implementiert in reva (feature/gateway-listcontainerstream).
Gateway leitet den Stream vom Storage Provider durch.

### 5. Metadata-Change-Event (Prio: niedrig, separates Thema)

SetArbitraryMetadata emittet kein Event → Search-Index wird nicht aktualisiert.
Gilt für WebDAV PROPPATCH genauso. Separates Issue.

## Implementierungsreihenfolge

1. **ListContainerStream Gateway** — PR fertig, muss getestet werden
2. **BatchStat** — größter Impact für VFS/Sync-Clients
3. **BatchInitiateFileDownload** — reduziert Hydration-Latenz
4. **BatchSetArbitraryMetadata** — für Bulk-Metadata-Operationen

## Betroffene Repos

- cs3org/cs3apis (Protobuf-Definitionen)
- opencloud-eu/reva (Gateway + Storage Provider Implementierung)
- opencloud-eu/opencloud (ggf. Config, wenn neue HTTP-Endpoints)
- kosmos-eu/opencloud_cloudd (erster Client der Batch nutzen würde)

## Referenzen

- WebDAV MULTIGET RFC-Draft: draft-reschke-webdav-multiget (CalDAV/CardDAV)
- CS3 API Docs: buf.build/cs3org-buf/cs3apis
- cloudd Repo: codeberg.org/kosmos-eu/opencloud_cloudd
