# Homelab Media

This context names the media collection exposed by the homelab and the artifacts that may be managed beside it.

## Language

**Media Source**:
An existing directory tree whose media files are indexed in place by a media server; files are not copied or relocated as part of indexing.
_Avoid_: Import folder, ingestion directory

**Sidecar Metadata**:
Artwork, metadata documents, and related files stored beside the media they describe and writable by the media server.
_Avoid_: Media database, cache

**Central Media Metadata**:
The media server's database, scraped descriptions, and artwork stored independently from the media sources under application storage.
_Avoid_: Sidecar metadata, embedded metadata
