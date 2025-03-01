# Terradock OSM Tile Server

A modernized OpenStreetMap tile server environment using Ubuntu 24.04 and PostgreSQL 16. This project builds upon the original [OpenStreetMap website repository](https://github.com/openstreetmap/openstreetmap-website) with updates to support the latest Ubuntu and PostgreSQL versions.

## Overview

Terradock OSM Tile Server provides a containerized environment for running your own OpenStreetMap tile server with updated dependencies:

- Base OS: Ubuntu 24.04 LTS
- Database: PostgreSQL 16 with PostGIS extensions
- Rendering stack: Mapnik, mod_tile, and renderd
- Data management using osm2pgsql

## Features

- **Modern stack**: Uses the latest Ubuntu 24.04 LTS and PostgreSQL 16
- **Containerized**: Easy deployment with Docker
- **Customizable**: Configure your own map styles and rendering rules
- **Scalable**: Supports various deployment sizes from personal to production
- **Complete solution**: Includes the entire OpenStreetMap stack for tile generation

## Prerequisites

- Docker and Docker Compose
- At least 4GB RAM (8GB+ recommended for larger datasets)
- Sufficient disk space for OSM data (varies by region)

## Quick Start

1. Clone this repository:
   ```
   git clone https://github.com/Sriram-PR/terradock-osm-tile-server.git
   cd terradock-osm-tile-server
   ```

2. Build the Docker image:
   ```
   docker build -t terradock-osm-tile-server .
   ```

3. Create Docker volumes for persistent data:
   ```
   docker volume create osm-data-new
   docker volume create osm-tiles-new
   ```

4. Import OpenStreetMap data (using Luxembourg as an example):
   ```
   docker run `
       -e DOWNLOAD_PBF="https://download.geofabrik.de/europe/luxembourg-latest.osm.pbf" `
       -v osm-data-new:/data/database/ `
       -v osm-tiles-new:/data/tiles/ `
       terradock-osm-tile-server `
       import
   ```

5. Run the tile server:
   ```
   docker run `
       -p 8080:80 `
       -v osm-data-new:/data/database/ `
       -v osm-tiles-new:/data/tiles/ `
       -d terradock-osm-tile-server `
       run
   ```

6. Access the web interface at `http://localhost:8080`

## Architecture

The system consists of several containerized components:

- **Database**: PostgreSQL 16 with PostGIS for storing OSM data
- **Import**: Tools for importing and updating OSM data
- **Renderer**: Map tile rendering service
- **Web**: Front-end web server for displaying maps

## Performance Tuning

### PostgreSQL Optimization

The PostgreSQL 16 configuration has been optimized for OSM data with the following settings:

- Increased shared_buffers (25% of available RAM)
- Optimized work_mem for spatial queries
- Adjusted autovacuum settings for OSM data patterns
- Enhanced GiST index settings for spatial data

Edit `postgresql.conf` to further tune according to your hardware.

### Rendering Optimization

The rendering stack is configured with:

- Multi-threaded rendering
- Optimized cache settings
- Memory-tuned Mapnik parameters

## Production Deployment

For production environments, consider:

1. Setting up proper backup routines for the PostgreSQL database
2. Implementing a CDN for tile caching
3. Adding monitoring and alerting
4. Configuring SSL for secure connections

## Troubleshooting

### Common Issues

- **Renderer crashes**: Check available memory and increase if needed
- **Database connection errors**: Verify PostgreSQL settings and user credentials
- **Missing tiles**: Check rendering logs and ensure data was imported correctly

## Differences from Original Project

This repository differs from the original OpenStreetMap website in several ways:

1. Updated to Ubuntu 24.04 LTS (from older Ubuntu versions)
2. PostgreSQL upgraded to version 16 with optimized settings
3. Modernized containerization approach
4. Simplified deployment process
5. Performance improvements for modern hardware

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

## License

This project is licensed under the [Apache-2.0 License](https://github.com/Sriram-PR/terradock-osm-tile-server/blob/main/LICENSE).

## Acknowledgments

- [OpenStreetMap project](https://www.openstreetmap.org/) for the original software
- All contributors to the OpenStreetMap ecosystem
- The PostgreSQL and PostGIS communities