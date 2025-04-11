# Plex-CleanOnDeck
Simple, interactive bash script which cleans your "On deck" plex items.

# Usage:

Usage: ./CleanOnDeck.sh <LibrarySectionID>

You can get the section ID's by checking the "Get Info" from any media file through Plex, within the XML....


# Configuration:
PLEX_URL="http://192.168.1.100:32400"   # Replace with your Plex server IP

TOKEN="-YOUR-TOKEN-GOES-HERE-"          # Replace with your actual Plex token

# Requirements:

sudo apt install libxml2-utils
