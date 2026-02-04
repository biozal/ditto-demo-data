# Ditto Demo Data

This repository contains sample data designed for import into [Ditto Portal](https://portal.ditto.live/) for demonstrations and testing.

## What is Ditto?

Ditto is an edge-first, offline-first distributed database designed for mobile and edge applications. It enables devices to sync data peer-to-peer via Bluetooth, Wi-Fi, or LAN without requiring a central server.

## Available Demo Datasets

### Sample Logistics (`pubsec\sample-logistics\`)

**Recommended demo dataset** - An aircrew digital coordination system showcasing:
- Offline-first operations in low-connectivity environments
- Real-time collaboration between flight crews and operations
- Subscription-driven sync with location-based filtering
- Storage management with eviction strategies

**📊 View the interactive ERD:** Open `pubsec\sample-logistics\ERD.html` in your browser to see the complete entity relationship diagram with:
- Data hierarchy visualization
- Subscription patterns by role
- Eviction strategies
- Field-level documentation

**Documentation Files:**
- `ERD.html` - Interactive visual entity relationship diagram (open in browser)
- `SCHEMA.md` - Complete schema design with field-by-field documentation
- `SUBSCRIPTION_PATTERNS.md` - Guide to subscription-driven design patterns
- `DEMO_QUERIES.md` - 50+ example DQL queries for demonstrations

**Data Files:**
- `crew_members.json` - 50 crew members (long-lived reference data)
- `aircraft.json` - 15 aircraft (long-lived reference data)
- `missions.json` - 20 missions (short-lived transactional data)
- `crew_assignments.json` - 64 crew assignments
- `flight_logs.json` - 23 flight logs
- `mission_briefs.json` - 24 mission briefs
- `status_updates.json` - 71 status updates
- `maintenance_status.json` - 25 maintenance records

### Warehouse Operations (`warehouse\`)

Original demo dataset focused on warehouse operations and parts inventory management across logistics centers.

**Data Files:**
- `locations.json` - 125 locations (25 bases + 100 buildings)
- `people.json` - 2,500 personnel records

## Importing Data into Ditto Portal

1. Log in to [Ditto Portal](https://portal.ditto.live/)
2. Navigate to your app
3. Select the collection you want to populate
4. Import the corresponding JSON file

## Data Characteristics

All sample data follows Ditto best practices:
- Small, flat document structures (< 250 KB per document)
- No arrays (uses maps for mergeable concurrent updates)
- Foreign key references for relationships
- Subscription-friendly field design
- Eviction-ready with timestamps and status fields

## Additional Resources

- [Ditto Documentation](https://docs.ditto.live/)
- [Data Modeling Best Practices](https://docs.ditto.live/best-practices/data-modeling)
- [Ditto Portal](https://portal.ditto.live/)

## License

MIT License - see [LICENSE](LICENSE) file for details.
