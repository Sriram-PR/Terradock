# TerraDock: A Modernized OpenStreetMap Tile Server

A containerized OpenStreetMap tile server environment built on Ubuntu 24.04 and PostgreSQL 16. This project modernizes the original OSM tile server implementation with updated components and optimized configurations.

## Overview

Terradock provides a complete solution for running your own OpenStreetMap tile server with the latest stack:

- **Base OS**: Ubuntu 24.04 LTS
- **Database**: PostgreSQL 16 with PostGIS 3
- **Rendering**: Mapnik, mod_tile, and renderd
- **Import Tools**: osm2pgsql and osmosis
- **Web Interface**: Leaflet-based viewer

## Features

- **Modern Stack**: Uses the latest Ubuntu 24.04 LTS and PostgreSQL 16
- **Multi-stage Docker Build**: Optimized container size and build process
- **Configurable Rendering**: Customize threads, styles, and database settings
- **Regional Data Support**: Import specific regions and filter updates
- **Automatic Updates**: Optional hourly updates from OpenStreetMap
- **Performance Optimized**: Database and rendering configurations tuned for modern hardware
- **Flexible Deployment**: Suitable for personal use through production environments

## Prerequisites

- Docker and Docker Compose
- At least 4GB RAM (8GB+ recommended)
- Sufficient disk space for OSM data:
  - Small region: 20GB+
  - Country: 100GB+
  - Continent/Global: 500GB+

## Quick Start

1. Clone this repository:
   ```bash
   git clone https://github.com/Sriram-PR/terradock-osm-tile-server.git
   cd terradock-osm-tile-server
   ```

2. Build the Docker image:
   ```bash
   docker build -t terradock .
   ```

3. Create Docker volumes for persistent data:
   ```bash
   docker volume create osm-data
   docker volume create osm-tiles
   ```

4. Import OpenStreetMap data (Luxembourg example):
   ```bash
   docker run \
       -e DOWNLOAD_PBF="https://download.geofabrik.de/europe/luxembourg-latest.osm.pbf" \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       terradock \
       import
   ```

5. Run the tile server:
   ```bash
   docker run \
       -p 8080:80 \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       -d terradock \
       run
   ```

6. Access the web interface at `http://localhost:8080`

## Using Your Own Data

Instead of downloading data, you can use your own OSM PBF file:

```bash
docker run \
    -v /absolute/path/to/your-region.osm.pbf:/data/region.osm.pbf \
    -v osm-data:/data/database/ \
    -v osm-tiles:/data/tiles/ \
    terradock \
    import
```

For regional updates, also include the corresponding polygon file:

```bash
docker run \
    -e UPDATES=enabled \
    -v /absolute/path/to/your-region.osm.pbf:/data/region.osm.pbf \
    -v /absolute/path/to/your-region.poly:/data/region.poly \
    -v osm-data:/data/database/ \
    -v osm-tiles:/data/tiles/ \
    terradock \
    import
```

## Configuration Options

### Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `THREADS` | Number of threads for importing/rendering | 4 |
| `UPDATES` | Enable automatic updates | disabled |
| `ALLOW_CORS` | Enable CORS headers | disabled |
| `FLAT_NODES` | Use flat nodes file for large imports | disabled |
| `PGPASSWORD` | PostgreSQL password for renderd user | renderd |
| `NAME_LUA` | Custom Lua script name | openstreetmap-carto.lua |
| `NAME_STYLE` | Custom style file name | openstreetmap-carto.style |
| `NAME_MML` | Custom CartoCSS project file | project.mml |
| `NAME_SQL` | Custom SQL indexes file | indexes.sql |
| `DOWNLOAD_PBF` | URL to download OSM PBF data | NULL |
| `DOWNLOAD_POLY` | URL to download region polygon | NULL |
| `AUTOVACUUM` | Control PostgreSQL autovacuum | on |
| `OSM2PGSQL_EXTRA_ARGS` | Additional args for osm2pgsql | "" |
| `REPLICATION_URL` | URL for updates | https://planet.openstreetmap.org/replication/hour/ |
| `MAX_INTERVAL_SECONDS` | Max time between updates | 3600 |
| `EXPIRY_MINZOOM` | Minimum zoom for tile expiry | 13 |
| `EXPIRY_TOUCHFROM` | Minimum zoom to mark tiles dirty | 13 |
| `EXPIRY_DELETEFROM` | Minimum zoom to delete tiles | 19 |
| `EXPIRY_MAXZOOM` | Maximum zoom for tile expiry | 20 |

### Resource Requirements

Resources needed vary based on your deployment size:

| Deployment | RAM | CPU | Storage | Example Use Case |
|------------|-----|-----|---------|------------------|
| Minimal | 4GB | 2 | 20GB | Small region/city |
| Recommended | 8GB | 4 | 100GB | Country-level |
| Production | 16GB+ | 8+ | 500GB+ | Continental/Global |

## Architecture

The system architecture consists of several integrated components:

- **PostgreSQL + PostGIS**: Spatial database storing OSM data
- **osm2pgsql**: Imports OpenStreetMap data into PostgreSQL
- **Osmosis**: Handles incremental updates from OpenStreetMap
- **Mapnik**: Rendering library that converts data to tiles
- **mod_tile/renderd**: Manages tile rendering and caching
- **Apache**: Web server that delivers tiles and the viewer interface

## Volume Structure

The container uses volumes mounted at specific paths:

```
/data/
├── database/          # PostgreSQL database files and flat nodes
│   ├── postgres/      # PostgreSQL data directory
│   ├── flat_nodes.bin # Optional flat nodes file for large imports
│   └── region.poly    # Optional polygon file for regional updates
├── style/             # Stylesheet files
│   ├── mapnik.xml     # Compiled Mapnik XML style
│   ├── project.mml    # CartoCSS project file
│   └── *.style/*.lua  # osm2pgsql style files
├── tiles/             # Rendered tile cache
└── region.osm.pbf     # OSM data file to import
```

## Performance Tuning

### PostgreSQL Optimization

PostgreSQL 16 configuration is optimized with settings for different hardware profiles:

#### 8GB RAM System (Recommended)
```
shared_buffers = 2GB
work_mem = 256MB
maintenance_work_mem = 512MB
effective_cache_size = 4GB
max_parallel_workers = 8
```

#### 16GB+ RAM System (Production)
```
shared_buffers = 4GB
work_mem = 512MB
maintenance_work_mem = 1GB
effective_cache_size = 8GB
max_parallel_workers = 16
```

Edit the PostgreSQL template configurations for your specific hardware.

### Rendering Optimization

Rendering performance can be tuned via:

- `THREADS` environment variable to control rendering threads
- `OSM2PGSQL_EXTRA_ARGS="-C <cache size in MB>"` for import cache (default: 800MB)
- Tile expiry settings in the update script

## Using Custom Styles

To use a custom style, mount it into the `/data/style/` directory:

```bash
docker run -p 80:80 \
    -v /path/to/data:/data \
    -v /path/to/custom-style:/data/style \
    terradock run
```

Your style should include:
- CartoCSS project file (.mml)
- osm2pgsql style files (.style and .lua)
- Any additional resources like shapefiles or SQL scripts

## Update Process

When automatic updates are enabled (`UPDATES=enabled`):

1. The system downloads changeset files from OpenStreetMap's replication server
2. Changes are filtered to your region of interest (if a polygon is defined)
3. Updates are applied to the PostgreSQL database
4. Affected tiles are marked for re-rendering based on zoom level settings:
   - Zoom 13-18: Marked for re-rendering
   - Zoom 19-20: Deleted (to save space)

The update process runs automatically via cron. Enable updates when running the server:

```bash
docker run \
    -p 8080:80 \
    -e UPDATES=enabled \
    -e REPLICATION_URL=https://planet.openstreetmap.org/replication/minute/ \
    -e MAX_INTERVAL_SECONDS=60 \
    -v osm-data:/data/database/ \
    -v osm-tiles:/data/tiles/ \
    -d terradock \
    run
```

## Connecting to PostgreSQL

To connect to the PostgreSQL database:

```bash
docker run \
    -p 8080:80 \
    -p 5432:5432 \
    -v osm-data:/data/database/ \
    -d terradock \
    run
```

Connect using:
```bash
psql -h localhost -U renderd gis
```

The default password is `renderd` but can be changed using the `PGPASSWORD` environment variable.

## Production Deployment

For production environments:

1. **Scaling**:
   - Use higher `THREADS` values on multi-core systems
   - Consider separate database and rendering servers for large deployments

2. **Security**:
   - Set a strong `PGPASSWORD`
   - Use a reverse proxy with SSL termination
   - Restrict access to the PostgreSQL port

3. **High Availability**:
   - Implement a CDN for tile caching
   - Consider load balancing across multiple rendering servers
   - Set up regular database backups

4. **Monitoring**:
   - Monitor disk space for the database and tile cache
   - Track rendering queue length and performance
   - Set up alerts for service disruptions

## Using with Docker Compose

Create a `docker-compose.yml` file:

```yaml
version: '3'
services:
  terradock:
    build: .
    ports:
      - "8080:80"
    volumes:
      - osm-data:/data/database
      - osm-tiles:/data/tiles
    environment:
      - THREADS=4
      - UPDATES=enabled
    command: run

volumes:
  osm-data:
  osm-tiles:
```

Run with:
```bash
docker-compose up -d
```

## Troubleshooting

### Common Issues

| Issue | Possible Causes | Solution |
|-------|----------------|----------|
| Slow initial rendering | Normal for first-time tile generation | Tiles are cached after first rendering |
| Database connection errors | PostgreSQL not started or misconfigured | Check PostgreSQL logs and configuration |
| Missing fonts | Font packages not installed | Ensure all required font packages are installed |
| Out of memory | Insufficient RAM for dataset size | Increase available memory or reduce `shared_buffers` |
| Slow updates | Large update files or regional filtering | Check update logs for bottlenecks |
| "No space left on device" | Shared memory limit too low | Add `--shm-size="192m"` to docker run command |
| Import process crashes | Memory limitations | Try enabling `FLAT_NODES=enabled` or reduce cache size |

### Shared Memory Issues

If you encounter errors like "could not resize shared memory segment" or "No space left on device", increase the shared memory limit:

```bash
docker run \
    -p 8080:80 \
    -v osm-data:/data/database/ \
    --shm-size="192m" \
    -d terradock \
    run
```

### Logs

Important logs are available in the container at:

- `/var/log/tiles/run.log`: Main operation log
- `/var/log/tiles/osmosis.log`: Update download log
- `/var/log/tiles/osm2pgsql.log`: Database import log
- `/var/log/tiles/expiry.log`: Tile expiry log
- `/var/log/apache2/error.log`: Web server errors

## Special Use Cases

### Large Planet Imports

For importing the entire planet or very large regions:

```bash
docker run \
    -v /path/to/planet.osm.pbf:/data/region.osm.pbf \
    -v osm-data:/data/database/ \
    -e "FLAT_NODES=enabled" \
    -e "OSM2PGSQL_EXTRA_ARGS=-C 4096" \
    terradock \
    import
```

### Air-gapped Environments

For systems without internet access:
1. Import data on an internet-connected system
2. Export the volumes:
   ```bash
   docker run --rm -v osm-data:/data -v $(pwd):/backup alpine tar -czf /backup/osm-data.tar.gz /data
   docker run --rm -v osm-tiles:/data -v $(pwd):/backup alpine tar -czf /backup/osm-tiles.tar.gz /data
   ```
3. Transfer the tar files to the target system
4. Restore volumes:
   ```bash
   docker volume create osm-data
   docker volume create osm-tiles
   docker run --rm -v osm-data:/data -v $(pwd):/backup alpine tar -xzf /backup/osm-data.tar.gz -C /
   docker run --rm -v osm-tiles:/data -v $(pwd):/backup alpine tar -xzf /backup/osm-tiles.tar.gz -C /
   ```

## Apache Configuration

The tile server uses Apache with mod_tile to serve map tiles. The configuration file (`apache.conf`) contains:

```apache
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    AddTileConfig /tile/ default
    LoadTileConfigFile /etc/renderd.conf
    ModTileRenderdSocketName /run/renderd/renderd.sock
    ModTileRequestTimeout 0
    ModTileMissingRequestTimeout 30
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
    <IfDefine ALLOW_CORS>
        Header set Access-Control-Allow-Origin "*"
    </IfDefine>
</VirtualHost>
```

Key settings:
- **AddTileConfig**: Maps the `/tile/` URL path to the "default" renderd configuration
- **ModTileRenderdSocketName**: Specifies the socket used to communicate with the renderd daemon
- **ModTileRequestTimeout**: Set to 0 (no timeout) for rendering requests
- **ModTileMissingRequestTimeout**: 30-second timeout for missing tiles

## Cross-Origin Resource Sharing (CORS)

To enable CORS for serving tiles to other domains:

```bash
docker run \
    -p 8080:80 \
    -v osm-data:/data/database/ \
    -e ALLOW_CORS=enabled \
    -d terradock \
    run
```

This activates the `<IfDefine ALLOW_CORS>` section in the Apache configuration, adding the necessary CORS headers.

## Differences from Original Project

This repository improves upon the original OSM tile server in several ways:

1. **Modernized Stack**: Updated to Ubuntu 24.04 and PostgreSQL 16
2. **Optimized Configurations**: Better tuned for modern hardware
3. **Enhanced Containerization**: Multi-stage builds and proper volume management
4. **Improved Documentation**: Comprehensive setup and tuning guides
5. **Simplified Deployment**: Streamlined import and run processes

## Components

The project comprises several key files:

- **Dockerfile**: Multi-stage build for the container
- **postgresql.custom.conf.tmpl**: Database configuration template
- **openstreetmap-tiles-update-expire.sh**: Update and tile expiry script
- **run.sh**: Main entry script controlling import and run modes
- **apache.conf**: Apache web server configuration for tile serving
- **leaflet-demo.html**: Web interface for viewing tiles

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the [Apache-2.0 License](https://github.com/Sriram-PR/terradock-osm-tile-server/blob/main/LICENSE).

## Acknowledgments

- [OpenStreetMap project](https://www.openstreetmap.org/) for the original software
- All contributors to the OpenStreetMap ecosystem
- The PostgreSQL and PostGIS communities
