# Terradock: A Modernized OpenStreetMap Tile Server

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

### Option 1: Using Pre-built Image from GitHub Container Registry

1. Pull the image from GitHub Container Registry:
   ```bash
   docker pull ghcr.io/sriram-pr/terradock:latest
   ```

2. Create Docker volumes for persistent data:
   ```bash
   docker volume create osm-data
   docker volume create osm-tiles
   ```

3. Import OpenStreetMap data (Luxembourg example):
   ```bash
   docker run \
       -e DOWNLOAD_PBF="https://download.geofabrik.de/europe/luxembourg-latest.osm.pbf" \
       -e DOWNLOAD_POLY="https://download.geofabrik.de/europe/luxembourg.poly" \
       -e UPDATES=enabled \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       ghcr.io/sriram-pr/terradock:latest \
       import
   ```

4. Run the tile server:
   ```bash
   docker run \
       -p 8080:80 \
       -e UPDATES=enabled \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       -d ghcr.io/sriram-pr/terradock:latest \
       run
   ```

5. Access the web interface at `http://localhost:8080`

### Option 2: Building from Source

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
       -e DOWNLOAD_POLY="https://download.geofabrik.de/europe/luxembourg.poly" \
       -e UPDATES=enabled \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       terradock \
       import
   ```

5. Run the tile server:
   ```bash
   docker run \
       -p 8080:80 \
       -e UPDATES=enabled \
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

For regional updates, include the corresponding polygon file (highly recommended):

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

> **Important**: The polygon file is crucial for efficient updates, as it filters OSM changes to just your region of interest. Without it, your server will process all global changes, which can be extremely resource-intensive.

## Configuration Options

### Environment Variables

| Variable | Phase | Description | Default |
|----------|-------|-------------|---------|
| `THREADS` | Both | Number of threads for importing/rendering | 4 |
| `UPDATES` | Both | Enable automatic updates (must be set in both import and run) | disabled |
| `FLAT_NODES` | Both | Use flat nodes file for large imports | disabled |
| `PGPASSWORD` | Both | PostgreSQL password for renderd user | renderd |
| `ALLOW_CORS` | Run | Enable CORS headers | disabled |
| `NAME_LUA` | Import | Custom Lua script name | openstreetmap-carto.lua |
| `NAME_STYLE` | Import | Custom style file name | openstreetmap-carto.style |
| `NAME_MML` | Import | Custom CartoCSS project file | project.mml |
| `NAME_SQL` | Import | Custom SQL indexes file | indexes.sql |
| `DOWNLOAD_PBF` | Import | URL to download OSM PBF data | NULL |
| `DOWNLOAD_POLY` | Import | URL to download region polygon | NULL |
| `WGET_ARGS` | Import | Additional arguments for wget | NULL |
| `PG_VERSION` | Both | PostgreSQL version | 16 |
| `AUTOVACUUM` | Both | Control PostgreSQL autovacuum | on |
| `OSM2PGSQL_EXTRA_ARGS` | Import | Additional args for osm2pgsql | "" |
| `REPLICATION_URL` | Both | URL for updates | https://planet.openstreetmap.org/replication/hour/ |
| `MAX_INTERVAL_SECONDS` | Both | Max time between updates | 3600 |
| `EXPIRY_MINZOOM` | Run | Minimum zoom for tile expiry | 13 |
| `EXPIRY_TOUCHFROM` | Run | Minimum zoom to mark tiles dirty | 13 |
| `EXPIRY_DELETEFROM` | Run | Minimum zoom to delete tiles | 19 |
| `EXPIRY_MAXZOOM` | Run | Maximum zoom for tile expiry | 20 |

> **Note**: Variables marked as "Both" should be set identically during both import and run phases for proper operation.

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

## Key Parameter Usage Guide

Understanding when and how to use key parameters is crucial for optimal operation:

### THREADS
- **Import Phase**: Controls number of parallel processes for osm2pgsql database import
  ```bash
  -e THREADS=8 terradock import
  ```
- **Run Phase**: Controls number of rendering threads for renderd tile generation
  ```bash
  -e THREADS=8 terradock run
  ```
- **Best Practice**: Set this in BOTH phases, but can use different values based on workload (higher for import, lower for rendering if needed)

### FLAT_NODES
- **Import Phase**: Enables the flat nodes file for efficient memory usage during large imports
  ```bash
  -e FLAT_NODES=enabled terradock import
  ```
- **Run Phase**: Ensures the system knows where to find the flat nodes file
  ```bash
  -e FLAT_NODES=enabled terradock run
  ```
- **Best Practice**: Set this in BOTH phases if you use it at all, and keep it consistent

### OSM2PGSQL_EXTRA_ARGS
- **Import Phase ONLY**: Provides additional arguments to osm2pgsql during import
  ```bash
  -e OSM2PGSQL_EXTRA_ARGS="-C 4096 --drop" terradock import
  ```
- **Common Uses**:
  - `-C <cache size>`: Memory cache for import (larger = faster)
  - `--drop`: Drop tables before import (for reimporting)
  - `--slim`: Keep temporary tables (default in script)
  - `--hstore-all`: Store all tags in the hstore column
- **Best Practice**: Only needed during import, not used during run phase

### Summary

| Parameter | Import | Run | Notes |
|-----------|--------|-----|-------|
| THREADS | ✅ | ✅ | Set in both, but values can differ |
| FLAT_NODES | ✅ | ✅ | Must be consistent if used |
| OSM2PGSQL_EXTRA_ARGS | ✅ | ❌ | Import phase only |

For large imports (country or larger), a recommended configuration would be:
```bash
# Import phase
docker run -e THREADS=12 -e FLAT_NODES=enabled -e OSM2PGSQL_EXTRA_ARGS="-C 4096" terradock import

# Run phase
docker run -e THREADS=8 -e FLAT_NODES=enabled terradock run
```

## Performance Tuning

### PostgreSQL Optimization

PostgreSQL 16 configuration uses hardcoded settings in the template file. Consider modifying these values based on your hardware:

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

Edit the `postgresql.custom.conf.tmpl` file and rebuild the image for your specific hardware.

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

The update process runs automatically via cron. **Important**: The `UPDATES` flag must be set during both import and run phases:

```bash
# During import - sets up the initial osmosis workspace
docker run \
    -e UPDATES=enabled \
    -v osm-data:/data/database/ \
    -v osm-tiles:/data/tiles/ \
    terradock \
    import

# During run - activates the cron job
docker run \
    -p 8080:80 \
    -e UPDATES=enabled \
    -v osm-data:/data/database/ \
    -v osm-tiles:/data/tiles/ \
    -d terradock \
    run
```

To use a different update frequency:

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
| Update script fails | Path mismatch in update script | Verify that the script uses the correct path to trim_osc.py |

### Flat Nodes for Large Imports

For importing very large regions or the entire planet, use the `FLAT_NODES` option to improve performance and reduce memory requirements:

```bash
docker run \
    -v /path/to/planet.osm.pbf:/data/region.osm.pbf \
    -v osm-data:/data/database/ \
    -e "FLAT_NODES=enabled" \
    -e "OSM2PGSQL_EXTRA_ARGS=-C 4096" \
    terradock \
    import
```

When using `FLAT_NODES=enabled`, ensure you set it for both import and run phases.

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

## Long-term Maintenance

### Tile Cache Management

The tile cache grows continuously as users request new areas and zoom levels. Without proper management, it can consume hundreds of gigabytes or even terabytes of disk space:

```bash
# Check current tile cache size
docker exec -it [container_name] du -sh /data/tiles

# Clear the entire tile cache (use with caution)
docker exec -it [container_name] rm -rf /data/tiles/*
```

For selective cache management, consider these strategies:

1. **Age-Based Pruning**: Remove tiles older than a certain date:
   ```bash
   docker exec -it [container_name] find /data/tiles -type f -mtime +90 -delete
   ```

2. **Zoom-Level Pruning**: Remove only high-zoom tiles that are less frequently accessed:
   ```bash
   docker exec -it [container_name] find /data/tiles -path "*/[17-20]/*" -type f -delete
   ```

3. **Scheduled Maintenance**: Add a cron job to periodically manage the cache:
   ```bash
   0 3 * * 0 docker exec [container_name] find /data/tiles -type f -mtime +60 -delete
   ```

### Database Maintenance

While PostgreSQL's autovacuum handles basic maintenance, long-running OSM databases benefit from additional care:

1. **Manual VACUUM**: Run a full vacuum analyze periodically:
   ```bash
   docker exec -it [container_name] sudo -u postgres psql -d gis -c "VACUUM ANALYZE;"
   ```

2. **Reindex**: Rebuild indexes for better performance:
   ```bash
   docker exec -it [container_name] sudo -u postgres psql -d gis -c "REINDEX DATABASE gis;"
   ```

3. **Scheduled Maintenance Script**:
   ```bash
   #!/bin/bash
   # Run weekly database maintenance
   docker exec [container_name] sudo -u postgres psql -d gis -c "VACUUM ANALYZE;"
   # Monthly reindexing
   if [ $(date +%d) -eq "01" ]; then
     docker exec [container_name] sudo -u postgres psql -d gis -c "REINDEX DATABASE gis;"
   fi
   ```

4. **PostgreSQL Tuning**: For long-term operation, consider adjusting these settings in `postgresql.custom.conf.tmpl`:
   ```
   # Increase for large databases
   maintenance_work_mem = 1GB
   # More aggressive autovacuum
   autovacuum_vacuum_threshold = 1000
   autovacuum_analyze_threshold = 500
   ```

### Monitoring

Set up basic monitoring to ensure smooth operation:

1. **Disk Usage**:
   ```bash
   # Monitor database size
   docker exec [container_name] sudo -u postgres psql -d gis -c "SELECT pg_size_pretty(pg_database_size('gis'));"
   
   # Monitor tile cache growth
   docker exec [container_name] du -sh /data/tiles
   ```

2. **Rendering Queue**:
   ```bash
   # Check current rendering queue
   docker exec [container_name] sudo -u renderd renderd -t
   ```

3. **Update Lag**:
   ```bash
   # Check update lag (how far behind is your database)
   docker exec [container_name] /usr/bin/osmosis-db_replag
   ```

4. **Service Status**:
   ```bash
   # Check if services are running
   docker exec [container_name] service postgresql status
   docker exec [container_name] service apache2 status
   docker exec [container_name] ps aux | grep renderd
   ```

### Restart Procedures

Proper restart procedures prevent data corruption:

1. **Graceful Shutdown**:
   ```bash
   # Properly stop the container
   docker stop -t 120 [container_name]
   ```
   The extended timeout (120 seconds) allows PostgreSQL to shut down properly.

2. **After System Reboot**:
   ```bash
   # Start the container
   docker start [container_name]
   
   # Verify services
   docker exec [container_name] service postgresql status
   docker exec [container_name] service apache2 status
   docker exec [container_name] ps aux | grep renderd
   ```

3. **Recovery from Improper Shutdown**:
   If PostgreSQL won't start after an improper shutdown:
   ```bash
   # Run database recovery
   docker exec -it [container_name] sudo -u postgres pg_ctl -D /data/database/postgres/ recover
   docker exec -it [container_name] service postgresql start
   ```

### Version Upgrade Path

When upgrading to a new container version:

1. **Data Backup**:
   ```bash
   # Create a backup of your volumes
   docker run --rm -v osm-data:/data -v $(pwd):/backup alpine tar -czf /backup/osm-data-backup.tar.gz /data
   docker run --rm -v osm-tiles:/data -v $(pwd):/backup alpine tar -czf /backup/osm-tiles-backup.tar.gz /data
   ```

2. **Simple Upgrade** (for minor version changes):
   ```bash
   # Pull the new image
   docker pull ghcr.io/sriram-pr/terradock:latest
   
   # Stop and remove the old container
   docker stop [container_name]
   docker rm [container_name]
   
   # Start a new container with the same volumes
   docker run -p 8080:80 -v osm-data:/data/database -v osm-tiles:/data/tiles -d --name [container_name] ghcr.io/sriram-pr/terradock:latest run
   ```

3. **Database Migration** (for major PostgreSQL version changes):
   For significant version changes that require a database upgrade:
   ```bash
   # Export the database
   docker exec -it [old_container] sudo -u postgres pg_dump -Fc gis > gis_backup.dump
   
   # Start new container in import mode
   docker run -v $(pwd):/backup -v new-osm-data:/data/database -v new-osm-tiles:/data/tiles --name new-container ghcr.io/sriram-pr/terradock:latest import
   
   # Restore the database
   docker exec -it new-container sudo -u postgres pg_restore -d gis /backup/gis_backup.dump
   
   # Start in run mode
   docker stop new-container
   docker run -p 8080:80 -v new-osm-data:/data/database -v new-osm-tiles:/data/tiles -d --name new-container ghcr.io/sriram-pr/terradock:latest run
   ```

## Customization Guide

### Cron Job Customization

The default cron configuration runs the update script every minute (`* * * * *`), which is excessive for most deployments. To customize:

1. **Access the container**:
   ```bash
   docker exec -it [container_name] bash
   ```

2. **Edit the crontab**:
   ```bash
   nano /etc/crontab
   ```

3. **Modify the update frequency**:
   ```
   # Default (every minute)
   * * * * *   renderd    openstreetmap-tiles-update-expire.sh
   
   # Every 15 minutes
   */15 * * * *   renderd    openstreetmap-tiles-update-expire.sh
   
   # Hourly
   0 * * * *   renderd    openstreetmap-tiles-update-expire.sh
   
   # Every 6 hours
   0 */6 * * *   renderd    openstreetmap-tiles-update-expire.sh
   
   # Daily at 2 AM
   0 2 * * *   renderd    openstreetmap-tiles-update-expire.sh
   ```

4. **Restart cron**:
   ```bash
   service cron restart
   ```

5. **For persistence**, create a custom Dockerfile:
   ```dockerfile
   FROM ghcr.io/sriram-pr/terradock:latest
   COPY custom_crontab /etc/crontab
   ```

### Update Frequency Trade-offs

Choose the right update frequency based on your needs:

| Frequency | Configuration | CPU/IO Impact | Update Lag | Best For |
|-----------|---------------|--------------|------------|----------|
| Minute | `REPLICATION_URL=.../minute/` & cron: `* * * * *` | Very High | ~1 minute | Real-time apps, emergency services |
| 15 Minutes | `REPLICATION_URL=.../minute/` & cron: `*/15 * * * *` | High | ~15 minutes | Urban navigation, delivery services |
| Hourly | `REPLICATION_URL=.../hour/` & cron: `0 * * * *` | Medium | ~1 hour | General use, balanced approach |
| 6 Hours | `REPLICATION_URL=.../hour/` & cron: `0 */6 * * *` | Low | ~6 hours | Standard mapping applications |
| Daily | `REPLICATION_URL=.../day/` & cron: `0 2 * * *` | Very Low | ~1 day | Static maps, rural areas |

Example for hourly updates:
```bash
docker run -p 8080:80 \
    -e UPDATES=enabled \
    -e REPLICATION_URL=https://planet.openstreetmap.org/replication/hour/ \
    -e MAX_INTERVAL_SECONDS=3600 \
    -v osm-data:/data/database/ \
    -v osm-tiles:/data/tiles/ \
    -d terradock \
    run
```

### Expiry Settings Examples

Tile expiry settings control which zoom levels get re-rendered or deleted after updates:

| Setting | Description | Default |
|---------|-------------|---------|
| `EXPIRY_MINZOOM` | Minimum zoom to consider for expiry | 13 |
| `EXPIRY_TOUCHFROM` | Minimum zoom to mark as dirty | 13 |
| `EXPIRY_DELETEFROM` | Minimum zoom to delete instead of re-rendering | 19 |
| `EXPIRY_MAXZOOM` | Maximum zoom to consider for expiry | 20 |

Example configurations:

1. **Low-resource server** (minimize rendering load):
   ```bash
   -e EXPIRY_MINZOOM=13 \
   -e EXPIRY_TOUCHFROM=13 \
   -e EXPIRY_DELETEFROM=16 \
   -e EXPIRY_MAXZOOM=18
   ```
   This deletes higher zoom tiles (16+) instead of re-rendering them, reducing CPU load.

2. **High-performance server** (maximize quality):
   ```bash
   -e EXPIRY_MINZOOM=10 \
   -e EXPIRY_TOUCHFROM=10 \
   -e EXPIRY_DELETEFROM=20 \
   -e EXPIRY_MAXZOOM=20
   ```
   This re-renders all affected tiles from zoom 10 to 19, ensuring maximum freshness.

3. **Balanced approach**:
   ```bash
   -e EXPIRY_MINZOOM=12 \
   -e EXPIRY_TOUCHFROM=12 \
   -e EXPIRY_DELETEFROM=18 \
   -e EXPIRY_MAXZOOM=20
   ```

To implement these settings:

```bash
docker run -p 8080:80 \
    -e UPDATES=enabled \
    -e EXPIRY_MINZOOM=13 \
    -e EXPIRY_TOUCHFROM=13 \
    -e EXPIRY_DELETEFROM=18 \
    -e EXPIRY_MAXZOOM=20 \
    -v osm-data:/data/database/ \
    -v osm-tiles:/data/tiles/ \
    -d terradock \
    run
```

### Configuration Profiles for Different Scenarios

Here are recommended configurations for common use cases:

1. **Small City/Local Deployment** (minimal hardware, <4GB RAM):
   ```bash
   # Import
   docker run \
       -e THREADS=2 \
       -e DOWNLOAD_PBF="https://download.geofabrik.de/europe/luxembourg-latest.osm.pbf" \
       -e DOWNLOAD_POLY="https://download.geofabrik.de/europe/luxembourg.poly" \
       -e UPDATES=enabled \
       -e OSM2PGSQL_EXTRA_ARGS="-C 500" \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       terradock import

   # Run
   docker run -p 8080:80 \
       -e THREADS=2 \
       -e UPDATES=enabled \
       -e EXPIRY_DELETEFROM=16 \
       -e EXPIRY_MAXZOOM=18 \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       -d terradock run
   ```
   
   **Recommended cron**: Daily updates (`0 2 * * *`)

2. **Country-Level Deployment** (8GB RAM):
   ```bash
   # Import
   docker run \
       -e THREADS=4 \
       -e FLAT_NODES=enabled \
       -e DOWNLOAD_PBF="https://download.geofabrik.de/europe/germany-latest.osm.pbf" \
       -e DOWNLOAD_POLY="https://download.geofabrik.de/europe/germany.poly" \
       -e UPDATES=enabled \
       -e OSM2PGSQL_EXTRA_ARGS="-C 2048" \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       terradock import

   # Run
   docker run -p 8080:80 \
       -e THREADS=4 \
       -e FLAT_NODES=enabled \
       -e UPDATES=enabled \
       -e EXPIRY_MINZOOM=12 \
       -e EXPIRY_TOUCHFROM=12 \
       -e EXPIRY_DELETEFROM=18 \
       -e EXPIRY_MAXZOOM=19 \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       -d terradock run
   ```
   
   **Recommended cron**: Hourly updates (`0 * * * *`)

3. **Continental/Production Deployment** (16GB+ RAM):
   ```bash
   # Import
   docker run \
       -e THREADS=8 \
       -e FLAT_NODES=enabled \
       -e DOWNLOAD_PBF="https://download.geofabrik.de/europe-latest.osm.pbf" \
       -e DOWNLOAD_POLY="https://download.geofabrik.de/europe.poly" \
       -e UPDATES=enabled \
       -e OSM2PGSQL_EXTRA_ARGS="-C 8192 --number-processes 8" \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       terradock import

   # Run
   docker run -p 8080:80 \
       -e THREADS=6 \
       -e FLAT_NODES=enabled \
       -e UPDATES=enabled \
       -e EXPIRY_MINZOOM=10 \
       -e EXPIRY_TOUCHFROM=10 \
       -e EXPIRY_DELETEFROM=19 \
       -e EXPIRY_MAXZOOM=20 \
       -v osm-data:/data/database/ \
       -v osm-tiles:/data/tiles/ \
       --shm-size=512m \
       -d terradock run
   ```
   
   **Recommended cron**: 15-minute updates (`*/15 * * * *`)

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
6. **Consolidated Flat Nodes Handling**: Simplified approach to handling flat nodes

## Known Issues and Fixes

1. **PostgreSQL Configuration**: The PostgreSQL configuration has hardcoded memory settings that might not be optimal for all deployment environments. Consider modifying the `postgresql.custom.conf.tmpl` file for your specific hardware.

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