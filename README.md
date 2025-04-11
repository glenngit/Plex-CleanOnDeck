# Plex-CleanOnDeck
Simple, interactive bash script which cleans yoru "On deck" plex items.

Usage: Usage: ./CleanOnDeck.sh <LibrarySectionID>
You can get the section ID by checking the "Get Info" from any media file through Plex, within the XML....


 
#!/bin/bash

# CONFIGURATION
PLEX_URL="http://192.168.1.100:32400"   # Replace with your Plex server IP
TOKEN="-YOUR-TOKEN-GOES-HERE-"          # Replace with your actual Plex token

# ARGUMENT CHECK
SECTION_ID="$1"
if [ -z "$SECTION_ID" ]; then
  echo "Usage: $0 <LibrarySectionID>"
  exit 1
fi

# FETCH ON DECK ITEMS
FETCH_URL="$PLEX_URL/library/sections/$SECTION_ID/onDeck?X-Plex-Token=$TOKEN"
echo "➡️  Fetching URL: $FETCH_URL"
XML=$(curl -s "$FETCH_URL")

# PARSE AND EXTRACT <Video> ENTRIES
IFS=$'\n' read -d '' -r -a ITEMS < <(echo "$XML" | xmllint --xpath "//Video" - 2>/dev/null | sed -e 's/<Video /\n<Video /g' && printf '\0')

# MAP INDEXES TO ratingKeys
declare -A RATING_KEYS
i=1

echo ""
echo "📺 Items in 'Continue Watching':"
for ITEM in "${ITEMS[@]}"; do
  key=$(echo "$ITEM" | grep -o 'ratingKey="[0-9]*"' | cut -d'"' -f2)
  title=$(echo "$ITEM" | grep -o 'title="[^"]*"' | cut -d'"' -f2)

  # Skip entries without valid ratingKey or title
  if [ -z "$key" ] || [ -z "$title" ]; then
    continue
  fi

  type=$(echo "$ITEM" | grep -o 'type="[^"]*"' | cut -d'"' -f2)

  if [ "$type" == "episode" ]; then
    show=$(echo "$ITEM" | grep -o 'grandparentTitle="[^"]*"' | cut -d'"' -f2)
    season=$(echo "$ITEM" | grep -o 'parentIndex="[0-9]*"' | cut -d'"' -f2)
    episode=$(echo "$ITEM" | grep -o 'index="[0-9]*"' | cut -d'"' -f2)

    # Extra check to avoid empty episodes
    if [ -z "$show" ] || [ -z "$season" ] || [ -z "$episode" ]; then
      continue
    fi

    echo "$i) $show – S$(printf "%02d" "$season")E$(printf "%02d" "$episode") – $title"
  else
    year=$(echo "$ITEM" | grep -o 'year="[0-9]*"' | cut -d'"' -f2)
    echo "$i) $title (${year:-Unknown Year})"
  fi

  RATING_KEYS["$i"]="$key"
  ((i++))
done

# USER CHOICE
echo ""
read -p "Enter numbers to mark as watched (e.g. 1 3 5), or type 'all': " choice

# PROCESS CHOICE
if [ "$choice" == "all" ]; then
  for idx in "${!RATING_KEYS[@]}"; do
    SCRUB_URL="$PLEX_URL/:/scrobble?key=${RATING_KEYS[$idx]}&identifier=com.plexapp.plugins.library&X-Plex-Token=$TOKEN"
    echo "➡️  Marking item $idx as watched via: $SCRUB_URL"
    curl -s "$SCRUB_URL" > /dev/null
    echo "✔️  Done"
  done
else
  for idx in $choice; do
    key="${RATING_KEYS[$idx]}"
    if [ -n "$key" ]; then
      SCRUB_URL="$PLEX_URL/:/scrobble?key=$key&identifier=com.plexapp.plugins.library&X-Plex-Token=$TOKEN"
      echo "➡️  Marking item $idx as watched via: $SCRUB_URL"
      curl -s "$SCRUB_URL" > /dev/null
      echo "✔️  Done"
    else
      echo "❌ Invalid choice: $idx"
    fi
  done
fi

echo ""
echo "🎉 All selected items marked as watched!"
