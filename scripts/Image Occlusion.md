/*
# Image Occlusion for Excalidraw

This script creates image occlusion cards similar to Anki's Image Occlusion Enhanced plugin.

## Usage:
1. Insert an image into Excalidraw
2. Draw rectangles or ellipses over areas you want to occlude
3. Select the image and all shapes you want to use as masks
4. Run this script
5. Choose occlusion mode:
   - ⭐⠀      Add Cards:    Hide One, Guess One: Creates cards where only one shape is hidden at a time
   - ⭐⭐     Add Cards:    Hide All, Guess One: Creates cards where all shapes are hidden except one
   - 🗑️⠀      Delete Cards: Delete all old cards (only add DELETE marker): Marks all existing cards for deletion by adding DELETE marker
   - 🗑️💥     Delete Cards: Delete all old cards file and related images (Be Cautious!!): Permanently deletes all related card files and images

The script will generate masked versions of the image and save them locally.

```javascript
*/

// Check minimum required version of Excalidraw plugin
if(!ea.verifyMinimumPluginVersion || !ea.verifyMinimumPluginVersion("1.9.0")) {
  new Notice("This script requires a newer version of Excalidraw. Please install the latest version.");
  return;
}

// Get all selected elements from the canvas
const elements = ea.getViewSelectedElements();

// Find the image element among selected elements
const image = elements.find(el => el.type === "image");

// Get all non-image elements to use as masks
const maskElements = elements.filter(el => el.type !== "image");

// Group masks based on their grouping in Excalidraw
const maskGroups = ea.getMaximumGroups(maskElements);

// Process each mask or group of masks
const masks = maskGroups.map(group => {
  // If group contains only one element, return that element
  if (group.length === 1) return group[0];
  // If group contains multiple elements, return the group info
  return {
    type: "group",
    elements: group,
    id: group[0].groupIds?.[0] || ea.generateElementId()
  };
});

// Validate selection - must have one image and at least one mask
if(!image || masks.length === 0) {
  new Notice("Please select one image and at least one element or group to use as mask");
  return;
}

// Present user with operation mode choices
const mode = await utils.suggester(
  [
    "⭐⠀      Add Cards:    Hide One, Guess One",
    "⭐⭐     Add Cards:    Hide All, Guess One",
    "🗑️⠀      Delete Cards: Delete all old cards (only add DELETE marker)",
    "🗑️💥     Delete Cards: Delete all old cards file and related images (Be Cautious!!)"
  ],
  ["hideOne", "hideAll", "delete", "deleteFiles"],
  "Select operation mode"
);

// Exit if user cancels the operation
if(!mode) return;

// Function to permanently delete related files and images
const deleteRelatedFilesAndImages = async (sourcePath) => {
  // Initialize collections and counters
  const cardFiles = new Set();
  const batchMarkers = new Set();
  const sourceFile = app.vault.getAbstractFileByPath(sourcePath);
  let deletedCardsCount = 0;
  let deletedFoldersCount = 0;
  
  // Validate source file exists
  if (!sourceFile) {
    new Notice(`Source file not found: ${sourcePath}`);
    return;
  }
  
  // Get backlinks from metadata cache
  const resolvedLinks = app.metadataCache.resolvedLinks;
  
  // Find all files that link to the source file
  for (const [filePath, links] of Object.entries(resolvedLinks)) {
    if (links[sourcePath]) {
      let file = app.vault.getAbstractFileByPath(filePath);
      if (file) {
        // Read file content to determine its type
        const content = await app.vault.read(file);
        
        // Check if file is an Anki card by looking for START and END markers
        const lines = content.split('\n');
        const hasStart = lines.some(line => /^\s*START\s*$/.test(line));
        const hasEnd = lines.some(line => /^\s*END\s*$/.test(line));
        
        if (hasStart && hasEnd) {
          cardFiles.add(file);
          console.log(`Found card file to delete: ${file.path}`);
        }
        
        // Check if file is a batch marker file
        if (file.name === 'batch-marker.md') {
          batchMarkers.add(file);
          console.log(`Found batch-marker file: ${file.path}`);
        }
      }
    }
  }
  
  // First delete all card files
  for (const file of cardFiles) {
    try {
      // Verify file still exists before attempting deletion
      if (await app.vault.adapter.exists(file.path)) {
        await app.vault.delete(file);
        deletedCardsCount++;
        console.log(`Deleted card file: ${file.path}`);
      }
    } catch (error) {
      console.error(`Failed to delete card file: ${file.path}`, error);
    }
  }
  
  // Then delete batch marker folders after all cards are deleted
  for (const marker of batchMarkers) {
    // Get parent folder path from batch marker file path
    const parentPath = marker.path.substring(0, marker.path.lastIndexOf('/'));
    const parentFolder = app.vault.getAbstractFileByPath(parentPath);
    
    // Verify folder exists before attempting deletion
    if (parentFolder && await app.vault.adapter.exists(parentFolder.path)) {
      try {
        // Delete folder and all its contents recursively
        await app.vault.delete(parentFolder, true);
        deletedFoldersCount++;
        console.log(`Deleted folder: ${parentFolder.path}`);
      } catch (error) {
        console.error(`Failed to delete folder: ${parentFolder.path}`, error);
      }
    }
  }
  
  new Notice(`Summary:
  - Card files deleted: ${deletedCardsCount}
  - Image folders deleted: ${deletedFoldersCount}`);
};

// Function to find and mark cards for deletion
const deleteRelatedCards = async (sourcePath) => {
  const cardFiles = new Set();
  const sourceFile = app.vault.getAbstractFileByPath(sourcePath);
  let totalCardsFound = 0;
  let totalNewlyMarked = 0;
  let totalAlreadyMarked = 0;
  
  if (!sourceFile) {
    console.log(`Source file not found: ${sourcePath}`);
    return;
  }
  
  // Get resolved links (backlinks) from metadata cache
  const resolvedLinks = app.metadataCache.resolvedLinks;
  
  // Search through all files to find those linking to our source file
  for (const [filePath, links] of Object.entries(resolvedLinks)) {
    if (links[sourcePath]) {
      const file = app.vault.getAbstractFileByPath(filePath);
      if (file) {
        cardFiles.add(file);
        console.log(`Found related card file: ${file.path}`);
      }
    }
  }
  
  // Process each card file to add DELETE markers
  for (const file of cardFiles) {
    // Read file content and split into lines for processing
    const content = await app.vault.read(file);
    const lines = content.split('\n');
    let modified = false;
    let cardCount = 0;
    let alreadyMarkedCount = 0;
    
    // Search for Anki card IDs and add DELETE marker before each
    for (let i = 0; i < lines.length; i++) {
      // Look for Anki card ID pattern
      const idMatch = lines[i].match(/<!--ID: .+?-->/);
      if (idMatch) {
        cardCount++;
        const cardId = idMatch[0];
        
        // Check if DELETE marker already exists
        if (i > 0 && lines[i-1].trim() === 'DELETE') {
          alreadyMarkedCount++;
          continue;
        }
        
        // Insert DELETE marker before the ID line
        lines.splice(i, 0, 'DELETE');
        i++; // Skip the newly inserted line
        modified = true;
        console.log(`Added DELETE marker before ${cardId} in ${file.name}`);
      }
    }
    
    // Save changes if file was modified
    if (modified) {
      await app.vault.modify(file, lines.join('\n'));
    }
    
    totalCardsFound += cardCount;
    totalNewlyMarked += (cardCount - alreadyMarkedCount);
    totalAlreadyMarked += alreadyMarkedCount;
  }
  
  new Notice(`Summary:
  - Files processed: ${cardFiles.size}
  - Total cards found: ${totalCardsFound}
  - Newly marked for deletion: ${totalNewlyMarked}
  - Already marked for deletion: ${totalAlreadyMarked}`);
};

// If delete files mode is selected, delete all related files and exit
if(mode === "deleteFiles") {
  // Show confirmation dialog before permanent deletion
  const confirmed = await utils.suggester(
    ["Cancel", "Yes, permanently delete all files"],
    [false, true],
    "WARNING: This will permanently delete all related card files and image folders. This action cannot be undone. Are you sure?"
  );
  
  // Execute deletion if confirmed
  if (confirmed) {
    const currentFile = app.workspace.getActiveFile();
    if (currentFile) {
      await deleteRelatedFilesAndImages(currentFile.path);
    }
  } else {
    // User cancelled the operation
    new Notice("Operation cancelled");
  }
  return;
}

// If delete mode is selected, mark old cards for deletion and exit
if(mode === "delete") {
  const currentFile = app.workspace.getActiveFile();
  if (currentFile) {
    await deleteRelatedCards(currentFile.path);
  }
  return;
}

// Extract original image name from the file ID
const getImageName = (fileId) => {
  const imageData = ea.targetView.excalidrawData.getFile(fileId);
  if (imageData?.linkParts?.original) {
    const pathParts = imageData.linkParts.original.split('/');
    const fileName = pathParts[pathParts.length - 1];
    return fileName.split('.')[0]; // Remove extension
  }
  return 'image';
};

// Create timestamp in format: YYMMDDHHmmssSSS
const now = new Date();
const timestamp = now.getFullYear().toString().slice(-2) + 
                 (now.getMonth() + 1).toString().padStart(2, '0') +
                 now.getDate().toString().padStart(2, '0') +
                 now.getHours().toString().padStart(2, '0') +
                 now.getMinutes().toString().padStart(2, '0') +
                 now.getSeconds().toString().padStart(2, '0') +
                 now.getMilliseconds().toString().padStart(3, '0');

// Function to generate current timestamp for file names
const getCurrentTimestamp = () => {
  const now = new Date();
  const baseTimestamp = now.getFullYear().toString().slice(-2) + 
                         (now.getMonth() + 1).toString().padStart(2, '0') +
                         now.getDate().toString().padStart(2, '0') +
                         now.getHours().toString().padStart(2, '0') +
                         now.getMinutes().toString().padStart(2, '0') +
                         now.getSeconds().toString().padStart(2, '0') +
                         now.getMilliseconds().toString().padStart(3, '0');
  return baseTimestamp;
};

// Initialize or get script settings for card location
let settings = ea.getScriptSettings();
if(!settings["Card Location"]) {
  // Set default settings if not exists
  settings = {
    "Card Location": {
      value: "ask",
      description: "Where to save card files ('default' for same folder as images, or 'choose' for custom location)",
      valueset: ["ask"]
    }
  };
  await ea.setScriptSettings(settings);
}

// Function to prompt user for card file save location
const askForCardLocation = async (imageFolder) => {
  // Show location selection dialog
  const choice = await utils.suggester(
    ["Default location (with images)", "Choose custom location"],
    ["default", "custom"],
    "Where would you like to save the card files?"
  );
  
  // Return default location if no choice or default selected
  if(!choice || choice === "default") {
    return imageFolder;
  }
  
  // Get list of all available folders in vault
  const folders = app.vault.getAllLoadedFiles()
    .filter(f => f.children)
    .map(f => f.path)
    .sort();
  
  // Let user choose from available folders
  const selectedFolder = await utils.suggester(
    folders,
    folders,
    "Select folder for card files"
  );
  
  return selectedFolder || imageFolder;
};

// Function to construct image folder path using image name and timestamp
const getImageFolder = (imageName, timestamp) => {
  return `Excalidraw-Image-Occlusions/${imageName}-${timestamp}`;
};

// Function to determine final output folder path based on settings or user choice
const getOutputFolder = async (imageName, timestamp) => {
  // Get default image folder path
  const imageFolder = getImageFolder(imageName, timestamp);
  
  // Return default path if settings specify default location
  if(settings["Card Location"].value === "default") {
    return imageFolder;
  }
  
  // Get list of all available folders for user selection
  const folders = app.vault.getAllLoadedFiles()
    .filter(f => f.children)
    .map(f => f.path)
    .sort();
  
  // Let user choose output folder
  const selectedFolder = await utils.suggester(
    folders,
    folders,
    "Select folder for card files"
  );
  
  // Return default folder if no selection made
  if(!selectedFolder) {
    return imageFolder;
  }
  
  return selectedFolder;
};

// Create necessary folders for storing images and cards
const imageName = getImageName(image.fileId);
const imageFolder = getImageFolder(imageName, timestamp);
const cardFolder = await askForCardLocation(imageFolder);

// Create image folder with all parent directories
await app.vault.adapter.mkdir(imageFolder, { recursive: true });

// Create card folder if different from image folder
if(cardFolder !== imageFolder) {
  await app.vault.adapter.mkdir(cardFolder, { recursive: true });
}

// Function to convert base64 image data to binary format
const base64ToBinary = (base64) => {
  // Remove data URL prefix
  const base64Data = base64.replace(/^data:image\/png;base64,/, "");
  // Convert base64 to binary string
  const binaryString = window.atob(base64Data);
  // Convert binary string to Uint8Array
  const bytes = new Uint8Array(binaryString.length);
  for (let i = 0; i < binaryString.length; i++) {
    bytes[i] = binaryString.charCodeAt(i);
  }
  return bytes;
};

// Function to generate image with specified visible and hidden masks
const generateMaskedImage = async (visibleMasks = [], hiddenMasks = []) => {
  // Combine image and all masks into one array
  const allElements = [image];
  [...visibleMasks, ...hiddenMasks].forEach(mask => {
    if (mask.type === "group") {
      allElements.push(...mask.elements);
    } else {
      allElements.push(mask);
    }
  });
  // Copy elements to Excalidraw's editing area
  ea.copyViewElementsToEAforEditing(allElements);
  
  // Get and cache the original image data
  const imageData = ea.targetView.excalidrawData.getFile(image.fileId);
  if (imageData) {
    ea.imagesDict[image.fileId] = {
      id: image.fileId,
      dataURL: imageData.img,
      mimeType: imageData.mimeType,
      created: Date.now()
    };
  }

  // Configure visibility of masks for question image
  visibleMasks.forEach(mask => {
    if (mask.type === "group") {
      // Set all elements in group to fully visible
      mask.elements.forEach(el => {
        const element = ea.getElement(el.id);
        element.opacity = 100;
      });
    } else {
      // Set single element to fully visible
      const element = ea.getElement(mask.id);
      element.opacity = 100;
    }
  });

  // Configure invisibility of masks for answer image
  hiddenMasks.forEach(mask => {
    if (mask.type === "group") {
      // Set all elements in group to invisible
      mask.elements.forEach(el => {
        const element = ea.getElement(el.id);
        element.opacity = 0;
      });
    } else {
      // Set single element to invisible
      const element = ea.getElement(mask.id);
      element.opacity = 0;
    }
  });

  // Generate PNG with specific export settings
  const dataURL = await ea.createPNGBase64(
    null,
    1,
    {
      // Force light mode and white background for consistent images
      exportWithDarkMode: false,
      exportWithBackground: true,
      viewBackgroundColor: "#ffffff"
    }
  );

  // Clear Excalidraw's editing area
  ea.clear();
  return dataURL;
};

// Function to get available Templater templates
const getTemplates = () => {
  // Check if Templater plugin is installed
  const templaterPlugin = app.plugins.plugins["templater-obsidian"];
  if (!templaterPlugin) {
    new Notice("Templater plugin is not installed");
    return null;
  }
  
  // Check if template folder is configured
  const templateFolder = templaterPlugin.settings.templates_folder;
  if (!templateFolder) {
    new Notice("Template folder is not set in Templater settings");
    return null;
  }

  // Get template folder and verify it exists
  const templates = app.vault.getAbstractFileByPath(templateFolder);
  if (!templates || !templates.children) {
    new Notice("No templates found");
    return null;
  }

  // Return only markdown files from template folder
  return templates.children.filter(f => f.extension === "md");
};

// Function to create batch marker file with source link
const createSourceLinkFile = async (sourceFile) => {
  // Create content with link back to source file
  const content = `\n[[${sourceFile} |find edit source]]\n`;
  const fileName = `${imageFolder}/batch-marker.md`;
  await app.vault.create(fileName, content);
  return fileName;
};

// Function to create card markdown file from template
const createMarkdownFromTemplate = async (templatePath, cardNumber, imagePath, sourceFile) => {
  const templaterPlugin = app.plugins.plugins["templater-obsidian"];
  const template = await app.vault.read(templatePath);
  
  // Convert absolute file paths to relative paths for Obsidian links
  const vaultPath = app.vault.adapter.getBasePath();
  const relativePath = {
    question: imagePath.question.replace(vaultPath, '').replace(/\\/g, '/'),
    answer: imagePath.answer.replace(vaultPath, '').replace(/\\/g, '/')
  };
  
  // Replace template placeholders with actual values
  let content = template
    .replace(/{{card_number}}/g, cardNumber)
    .replace(/{{question}}/g, relativePath.question)
    .replace(/{{answer}}/g, relativePath.answer)
    .replace(/{{editSource}}/g, sourceFile)
    .replace(/{{batchMarker}}/g, `${imageFolder}/batch-marker.md`);
  
  // Create new card file with generated content
  const fileName = `${cardFolder}/${cardNumber}.card.md`;
  await app.vault.create(fileName, content);
};

// Helper function to get current Excalidraw file path
const getCurrentFilePath = () => {
  const file = app.workspace.getActiveFile();
  return file ? file.path : '';
};

// Begin card generation process based on selected mode
let counter = 1;
if(mode === "hideAll") {
  // Get template selection from user for Hide All mode
  const templates = getTemplates();
  if (!templates) return;
  
  // Let user select template file for card generation
  const templateFile = await utils.suggester(
    templates.map(t => t.basename),
    templates,
    "Select a template for the cards"
  );
  if (!templateFile) return;

  // Get path of current Excalidraw file for linking
  const sourceFile = getCurrentFilePath();

  // Create batch marker file to track related cards
  const batchMarkerFile = await createSourceLinkFile(sourceFile);

  // Generate cards for each mask in Hide All mode
  for(let i = 0; i < masks.length; i++) {
    // Set current mask as hidden, all others as visible
    const hiddenMasks = [masks[i]];
    const visibleMasks = masks.filter((_, index) => index !== i);
    
    // Generate unique timestamp for this card
    const fileTimestamp = getCurrentTimestamp();
    
    // Generate question image with all masks visible
    const questionDataURL = await generateMaskedImage(masks, []);
    const questionPath = `${imageFolder}/q-${fileTimestamp}.png`;
    await app.vault.adapter.writeBinary(
      questionPath,
      base64ToBinary(questionDataURL)
    );
    
    // Generate answer image with one mask hidden and others visible
    const dataURL = await generateMaskedImage(visibleMasks, hiddenMasks);
    const imagePath = `${imageFolder}/a-${fileTimestamp}.png`;
    
    // Save answer image to disk
    await app.vault.adapter.writeBinary(
      imagePath,
      base64ToBinary(dataURL)
    );

    // Create markdown file with full paths to images
    const fullPaths = {
      question: app.vault.adapter.getFullPath(questionPath),
      answer: app.vault.adapter.getFullPath(imagePath)
    };
    // Generate card file from template with all necessary information
    await createMarkdownFromTemplate(
      templateFile,
      fileTimestamp,
      fullPaths,
      sourceFile
    );
  }
} else {
  // Process Hide One, Guess One mode
  const templates = getTemplates();
  if (!templates) return;
  
  // Let user select template for card generation
  const templateFile = await utils.suggester(
    templates.map(t => t.basename),
    templates,
    "Select a template for the cards"
  );
  if (!templateFile) return;

  // Get current Excalidraw file path for linking
  const sourceFile = getCurrentFilePath();

  // Create batch marker file to track related cards
  const batchMarkerFile = await createSourceLinkFile(sourceFile);

  // Process each mask individually
  for(const mask of masks) {
    // Set current mask as visible, others as hidden for question
    const visibleMasks = masks.filter(m => m !== mask);
    const hiddenMasks = [mask];
    
    // Generate unique timestamp for this card
    const fileTimestamp = getCurrentTimestamp();
    
    // Generate question image showing only the current mask
    const questionDataURL = await generateMaskedImage([mask], visibleMasks);
    const questionPath = `${imageFolder}/q-${fileTimestamp}.png`;
    await app.vault.adapter.writeBinary(
      questionPath,
      base64ToBinary(questionDataURL)
    );
    
    // Generate answer image with all masks hidden
    const answerDataURL = await generateMaskedImage([], masks);
    const answerPath = `${imageFolder}/a-${fileTimestamp}.png`;
    await app.vault.adapter.writeBinary(
      answerPath,
      base64ToBinary(answerDataURL)
    );
    
    // Create markdown file with paths to question and answer images
    const fullPaths = {
      question: app.vault.adapter.getFullPath(questionPath),
      answer: app.vault.adapter.getFullPath(answerPath)
    };
    // Generate card file using template and image paths
    await createMarkdownFromTemplate(
      templateFile,
      fileTimestamp,
      fullPaths,
      sourceFile
    );
  }
}

// Display completion message with number of cards created
new Notice(`Generated ${masks.length} sets of files in ${imageFolder}/`); 
