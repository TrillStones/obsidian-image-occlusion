# Obsidian Image Occlusion

An Excalidraw script for creating Anki image occlusion cards in Obsidian, similar to Anki's Image Occlusion Enhanced add-on but integrated into your Obsidian workflow.

## Overview
This script allows you to create image occlusion flashcards directly in Obsidian using Excalidraw. It's designed to work seamlessly with your existing note-taking workflow while providing similar functionality to Anki's Image Occlusion Enhanced.

## Comparison with Anki's Image Occlusion Enhanced

### Advantages
1. **Integrated Workflow**
   - Create cards directly in Obsidian
   - Use Excalidraw's drawing tools
   - Keep source files with your notes
   - Edit cards without leaving Obsidian

2. **Enhanced Features**
   - Supports shape grouping
   - Maintains source file links
   - Batch operations for card management
   - Better file organization

3. **Version Control**
   - All files are plain text/images
   - Easy to backup and sync
   - Works with git and other VCS

### Limitations
1. **Editing Experience**
   - No real-time preview of occlusions
   - Must regenerate cards after edits
   - Cannot edit existing cards directly

2. **Card Types**
   - Only supports "Hide One" and "Hide All" modes
   - No support for "Hide All, Guess One by One"
   - Cannot mix different occlusion types in one card

3. **Performance**
   - Generating multiple cards takes longer
   - Manual sync with Anki required
   - Larger file size due to image storage

## Use Cases
- Creating anatomy flashcards
- Learning diagrams and charts
- Memorizing maps and locations
- Studying technical illustrations
- Learning character components



## Usage Instructions

### Prerequisites
1. Install required Obsidian plugins:
   - Excalidraw
   - Templater
   - Obsidian to Anki

2. Copy template files to your templates folder:
   ```
   templates/
   ├── Anki Card.md      # For image occlusion cards
   ```

### Creating Image Occlusion Cards
1. Open or create an Excalidraw file
2. Insert an image you want to study
3. Draw shapes over areas you want to occlude:
   - Use rectangles or ellipses
   - Group related shapes if needed
   - Position shapes precisely over target areas
4. Select both the image and all shapes you want to use as masks
5. Run the "Image Occlusion" script
6. Choose occlusion mode:
   - `⭐ Add Cards: Hide One, Guess One`
     - Creates one card per mask
     - Only one area is hidden at a time
   - `⭐⭐ Add Cards: Hide All, Guess One`
     - Creates cards where all areas are hidden except one
     - Tests recognition of individual items

### Managing Existing Cards
1. To mark cards for deletion:
   - Select source Excalidraw file
   - Run script and choose `🗑️ Delete Cards: Delete all old cards`
   - This adds DELETE marker before Anki card IDs
   - Sync with Anki to remove cards

2. To permanently delete files:
   - Select source Excalidraw file
   - Choose `🗑️💥 Delete Cards: Delete all old cards file and related images`
   - Confirm deletion
   - This removes:
     - All related card files
     - Generated images
     - Batch marker files

### File Organization
```
Excalidraw-Image-Occlusions/
└── image-name-timestamp/
    ├── batch-marker.md     # Links to source file
    ├── q-timestamp.png     # Question images
    ├── a-timestamp.png     # Answer images
    └── timestamp.card.md   # Card files
```

### Tips
- Draw precise masks around areas you want to study
- Use meaningful names for your Excalidraw files
- Group related shapes when they form a logical unit
- Review generated cards in Anki before bulk deletion
- Keep your source Excalidraw files organized

### Troubleshooting
- If cards don't appear in Anki:
  - Check Obsidian to Anki plugin settings
  - Verify card template format
  - Ensure proper sync with Anki
- If images don't display correctly:
  - Check file paths in card markdown
  - Verify image files exist in correct location
- For deletion issues:
  - Ensure you're selecting the correct source file
  - Check if files are properly linked
 

## ⚠️ Important Safety Notes

### Data Protection
1. **Always Backup Your Data**
   - Backup your vault before using deletion features
   - Keep copies of important Excalidraw files
   - Export your Anki deck regularly

2. **Deletion Risks**
   - `Delete Cards: Delete all old cards file and related images` is **IRREVERSIBLE**
   - This operation will:
     - Permanently delete all card files
     - Remove entire image folders
     - Delete batch marker files
   - No way to recover deleted files without backup

3. **Safe Deletion Practice**
   - Use `Delete Cards: Delete all old cards (only add DELETE marker)` first
   - Review marked cards in Anki
   - Sync with Anki to verify deletion
   - Only then consider using permanent deletion

4. **Before Using Permanent Deletion**
   - Double check selected source file
   - Verify which cards will be affected
   - Ensure you have backups
   - Consider using Obsidian's file recovery if available
   - Wait for Anki sync to complete

### Recommended Workflow
1. Regular Backups
   ```
   - Daily: Obsidian vault backup
   - Weekly: Anki deck export
   - Monthly: Full system backup
   ```

2. Safe Deletion Process
   ```
   1. Mark cards for deletion
   2. Review in Anki
   3. Sync with Anki
   4. Verify deletion
   5. Backup vault
   6. Run permanent deletion
   ```

3. Recovery Options
   - Keep backups for at least 30 days
   - Use version control if possible
   - Document your deletion operations
   - Test backup restoration periodically
