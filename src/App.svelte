<script lang="ts">
  /// <reference types="@webflow/designer-extension-typings" />
  import { onMount, tick } from "svelte";
  // Define Page type since it's not exported
  type Page = {
    id: string;
    type: string;
    getName(): Promise<string>;
    setName(name: string): Promise<void>;
    getSlug(): Promise<string>;
    setSlug(slug: string): Promise<void>;
    getParent(): Promise<any>;
    setParent(parent: any): Promise<void>;
    setMetadata(metadata: any): Promise<void>;
    getTitle(): Promise<string>;
    setTitle(title: string): Promise<void>;
    getDescription(): Promise<string>;
    setDescription(description: string): Promise<void>;
    isDraft(): Promise<boolean>;
    setDraft(isDraft: boolean): Promise<void>;
    usesTitleAsOpenGraphTitle(): Promise<boolean>;
    // Since there are many more methods we might not know about,
    // we'll use a type assertion in our code instead of listing them all
  } & Record<string, any>;

  // Import types from Webflow Designer Extension typings
  type Style = {
    getName(): Promise<string>;
    getProperties(): Promise<any>;
    id: string;
    setProperties(properties: any): Promise<null>;
    removeProperties(props: any[], options?: any): Promise<null>;
    getProperty(name: string): Promise<any>;
    setProperty(name: string, value: any): Promise<null>;
    removeProperty(name: string): Promise<null>;
    removeAllProperties(): Promise<null>;
  };

  type WebflowElement = {
    id: string;
    type: string;
    styles: boolean; // Indicates if element has styles
    getStyles: () => Promise<Style[]>;
    setStyles: (styles: Style[]) => Promise<void>;
  };

  type ElementId = {
    pageId: string;
    elementId: string;
    toString(): string;
  };

  // Define DynamoElement type for collection items
  type DynamoElement = {
    id: string;
    type: string;
    parent?: {
      id: string;
      type: string;
    };
    styles?: any[];
    getCollection?: () => Promise<any>;
    getStyles?: () => Promise<any[]>;
  };

  let stylesWithProperties: Array<{
    name: string;
    properties: any;
    breakpointProperties: any;
  }> = [];
  let duplicates: Array<string[]> = [];
  let propertiesByGroup: Array<string> = [];
  let loading = false;
  let searchTerm = "";
  let selectedGroup: number | null = null;

  let selectedDuplicates: string[] = [];
  let newStyleName = "";

  // New reactive progress info variable
  let progressInfo: string = "";
  // New reactive log messages array
  let logs: string[] = [];
  // Control if the log panel is visible
  let logPanelVisible: boolean = false;
  let isProcessing = false;
  let shouldStop = false;
  let currentOperation = "";
  let processedCount = 0;
  let totalCount = 0;
  let originalPage: any;
  let updatedCount = 0;
  // Track which pages have elements using our styles
  let pageMap = new Map<string, { pageName: string; elementCount: number }>();
  let totalElementsUpdated = 0;

  // Update the WebflowItem type definition to include more specific types
  type WebflowItem = {
    id: string;
    type: string;
    getName(): Promise<string>;
  };

  $: filteredDuplicates = duplicates.filter((group) => {
    if (searchTerm) {
      const index = duplicates.indexOf(group);
      const hasMatchingName = group.some((name) =>
        name.toLowerCase().includes(searchTerm.toLowerCase()),
      );
      const hasMatchingProperties = propertiesByGroup[index]
        .toLowerCase()
        .includes(searchTerm.toLowerCase());
      return hasMatchingName || hasMatchingProperties;
    }
    return true;
  });

  $: filteredPropertiesByGroup = searchTerm
    ? propertiesByGroup.filter((_, index) =>
        filteredDuplicates.includes(duplicates[index]),
      )
    : propertiesByGroup;

  // Predefined breakpoints supported by Webflow (see docs)
  const predefinedBreakpoints = [
    { id: "xxl" },
    { id: "xl" },
    { id: "large" },
    { id: "main" },
    { id: "medium" },
    { id: "small" },
    { id: "tiny" },
  ];

  // Mapping of breakpoint IDs to display names
  const breakpointDisplayNames: { [key: string]: string } = {
    main: "Main",
    xxl: "XXL",
    xl: "XL",
    large: "Large",
    medium: "Tablet",
    small: "Mobile Landscape",
    tiny: "Mobile Portrait",
  };

  const fetchStyles = async () => {
    loading = true;
    try {
      const styles = await webflow.getAllStyles();

      stylesWithProperties = await Promise.all(
        styles.map(async (style) => {
          const name = await style.getName();
          const bpProps = await getStylePropertiesForBreakpoints(style);
          return {
            name,
            properties: bpProps["main"], // Base (default) properties
            breakpointProperties: bpProps, // All breakpoints
          };
        }),
      );

      const result = detectDuplicates(stylesWithProperties);
      duplicates = result.duplicates;
      propertiesByGroup = result.propertiesByGroup;
    } catch (error) {
      console.error("Error scanning for duplicates:", error);
    } finally {
      loading = false;
    }
  };

  const detectDuplicates = (
    styles: Array<{ name: string; properties: any; breakpointProperties: any }>,
  ) => {
    const duplicates: string[][] = [];
    const propertiesByGroup: string[] = [];
    const processed = new Set<number>();

    for (let i = 0; i < styles.length; i++) {
      const {
        name: name1,
        properties: properties1,
        breakpointProperties: bpProps1,
      } = styles[i];
      if (processed.has(i) || Object.keys(properties1).length === 0) continue;
      const duplicateGroup = [name1];

      for (let j = i + 1; j < styles.length; j++) {
        if (processed.has(j)) continue;
        const {
          name: name2,
          properties: properties2,
          breakpointProperties: bpProps2,
        } = styles[j];
        if (JSON.stringify(properties1) === JSON.stringify(properties2)) {
          duplicateGroup.push(name2);
          processed.add(j);
        }
      }

      if (duplicateGroup.length > 1) {
        duplicates.push(duplicateGroup);
        propertiesByGroup.push(JSON.stringify(properties1, null, 2));
      }
      processed.add(i);
    }

    return { duplicates, propertiesByGroup };
  };

  const selectStyleInWebflow = async (styleName: string) => {
    try {
      const capabilities = await webflow.canForAppMode([
        webflow.appModes.canModifyStyles,
      ]);

      if (!capabilities.canModifyStyles) {
        await webflow.notify({
          type: "Error",
          message:
            "Please switch to Design mode and ensure you're on the main branch to modify styles.",
        });
        return;
      }

      const style = await webflow.getStyleByName(styleName);
      if (style) {
        const tempInput = document.createElement("input");
        tempInput.value = styleName;
        document.body.appendChild(tempInput);
        tempInput.select();
        document.execCommand("copy");
        document.body.removeChild(tempInput);

        await webflow.notify({
          type: "Info",
          message: `Style name "${styleName}" copied to clipboard. Press 'G' to open Style Panel and paste to search.`,
        });
      }
    } catch (error) {
      console.error(`Error finding style: ${styleName}`, error);
      await webflow.notify({
        type: "Error",
        message: "Failed to copy style name to clipboard.",
      });
    }
  };

  const removeStyle = async (styleName: string) => {
    try {
      // Get the style by name
      const style = await webflow.getStyleByName(styleName);
      if (!style) {
        console.log(`Style ${styleName} not found.`);
        return;
      }

      try {
        // Attempt to remove the style
        await webflow.removeStyle(style);
        console.log(`Style: ${styleName} was removed`);

        webflow.notify({
          type: "Success",
          message: `Successfully removed style "${styleName}"`,
        });

        await fetchStyles();
      } catch (err) {
        throw new Error(
          `Style "${styleName}" is still in use and cannot be removed.`,
        );
      }
    } catch (error) {
      console.error(`Error removing style: ${styleName}`, error);
      webflow.notify({
        type: "Error",
        message:
          error instanceof Error ? error.message : "Failed to remove style",
      });
    }
  };

  type ColorVariable = { id: string; type: "color" };
  type SizeVariable = { id: string; type: "size" };
  type StyleValue = { value: string };

  type WebflowPropertyValue =
    | string
    | ColorVariable
    | SizeVariable
    | StyleValue;

  type StyleProperties = Record<string, string | number | boolean | null>;
  type WebflowStyleProperties = {
    [key: string]: WebflowPropertyValue | undefined;
  };

  function isStyleValue(value: WebflowPropertyValue): value is StyleValue {
    return typeof value === "object" && "value" in value;
  }

  async function checkDesignerPermissions() {
    const capabilities = await webflow.canForAppMode([
      webflow.appModes.canModifyStyles,
      webflow.appModes.canDesign,
      webflow.appModes.canAccessCanvas,
    ]);

    if (
      !capabilities.canModifyStyles ||
      !capabilities.canDesign ||
      !capabilities.canAccessCanvas
    ) {
      throw new Error(
        "Please ensure you are in design mode and have permissions to modify styles.",
      );
    }
  }

  async function handleCreateStyle() {
    try {
      shouldStop = false;
      isProcessing = true;
      totalElementsUpdated = 0;
      logs = [
        `Starting to merge ${selectedDuplicates.length} styles into "${newStyleName}"...`,
      ];

      // Check permissions first
      await checkDesignerPermissions();

      // Store the current page before we start switching pages
      try {
        originalPage = await webflow.getCurrentPage();
        console.log("Stored original page:", await originalPage.getName());
      } catch (err) {
        console.warn("Failed to store original page:", err);
      }

      // 1. Get all selected styles first
      console.log(`🎯 Target styles to merge:`, selectedDuplicates);
      const targetStyles = await Promise.all(
        selectedDuplicates.map(async (name) => {
          const style = await webflow.getStyleByName(name);
          if (!style) throw new Error(`Style "${name}" not found`);
          console.log(`Found style: ${name} (ID: ${style.id})`);
          return style;
        }),
      );

      // 2. Create new style with properties from first style
      if (!targetStyles[0]) {
        throw new Error("No source style found");
      }

      let properties;
      try {
        properties = await targetStyles[0].getProperties();
        if (!properties) {
          throw new Error("Could not get properties from source style");
        }
      } catch (err: unknown) {
        const errorMessage =
          err instanceof Error ? err.message : "Unknown error occurred";
        throw new Error(`Failed to get properties from style: ${errorMessage}`);
      }

      const newStyle = await webflow.createStyle(newStyleName);
      if (!newStyle) {
        throw new Error("Failed to create new style");
      }

      try {
        await newStyle.setProperties(properties);
        console.log(`Created new style: ${newStyleName} (ID: ${newStyle.id})`);
        logs = [...logs, `Created new style "${newStyleName}"`];
      } catch (err: unknown) {
        // Try to clean up the failed style
        try {
          await webflow.removeStyle(newStyle);
        } catch {} // Ignore cleanup errors
        const errorMessage =
          err instanceof Error ? err.message : "Unknown error occurred";
        throw new Error(
          `Failed to set properties on new style: ${errorMessage}`,
        );
      }

      // 3. Get all pages - UPDATED to properly filter pages
      const pagesAndFolders = await webflow.getAllPagesAndFolders();
      const pages = pagesAndFolders.filter(
        (item: WebflowItem): item is Page =>
          item &&
          typeof item === "object" &&
          "type" in item &&
          item.type === "Page" &&
          "getName" in item &&
          typeof item.getName === "function",
      );

      console.log(`\n📄 Found ${pages.length} pages to scan`);
      totalCount = pages.length;
      processedCount = 0;
      logs = [...logs, `Scanning ${pages.length} pages...`];

      // 4. Check each page for elements using our styles
      for (const page of pages) {
        if (shouldStop) break;

        processedCount++;
        let pageName;
        try {
          pageName = await page.getName();
        } catch (err) {
          console.warn("Could not get page name:", err);
          pageName = `Page ${page.id}`;
        }

        console.log(`\n🔍 Scanning page: ${pageName}`);
        logs = [...logs, `Checking ${pageName}...`];

        try {
          await webflow.switchPage(page as any);
          const elements = await webflow.getAllElements();
          console.log(`Found ${elements.length} total elements`);

          // Group elements by CMS collection.
          // For elements with a collectionId, only keep the first encountered as representative.
          const collectionRepresentatives = new Map<string, any>();
          const standaloneElements = [];

          for (const element of elements) {
            if (!element?.styles) continue;

            const collectionId = await getCollectionTemplateId(element);
            if (collectionId) {
              if (!collectionRepresentatives.has(collectionId)) {
                collectionRepresentatives.set(collectionId, element);
              }
            } else {
              standaloneElements.push(element);
            }
          }

          let pageUpdatedCount = 0;

          // Process each CMS collection representative concurrently.
          await Promise.all(
            Array.from(collectionRepresentatives.entries()).map(
              async ([collectionId, repElement]) => {
                try {
                  const elementStyles = await repElement.getStyles();
                  if (!Array.isArray(elementStyles)) return;

                  console.log(
                    `\nChecking collection representative:`,
                    `\n  ID: ${repElement.id}`,
                    `\n  Type: ${repElement.type}`,
                    `\n  Collection: ${collectionId}`,
                    `\n  Styles: ${elementStyles.length}`,
                  );

                  const hasTargetStyle = elementStyles.some(
                    (style) =>
                      style &&
                      targetStyles.some((target) => target?.id === style?.id),
                  );

                  if (hasTargetStyle) {
                    const updatedStyles = elementStyles.map((style) =>
                      style &&
                      targetStyles.some((target) => target?.id === style?.id)
                        ? newStyle
                        : style,
                    );

                    await repElement.setStyles(
                      updatedStyles.filter((s): s is Style => s !== null),
                    );
                    pageUpdatedCount++;
                    logs = [
                      ...logs,
                      `Updated collection template ${collectionId}`,
                    ];
                  }
                } catch (err) {
                  console.warn(
                    `Error processing collection representative ${collectionId}:`,
                    err,
                  );
                }
              },
            ),
          );

          // Process standalone elements concurrently.
          await Promise.all(
            standaloneElements.map(async (element) => {
              try {
                const elementStyles = await element.getStyles();
                if (!Array.isArray(elementStyles)) return;

                const hasTargetStyle = elementStyles.some(
                  (style) =>
                    style &&
                    targetStyles.some((target) => target?.id === style?.id),
                );

                if (hasTargetStyle) {
                  const updatedStyles = elementStyles.map((style) =>
                    style &&
                    targetStyles.some((target) => target?.id === style?.id)
                      ? newStyle
                      : style,
                  );

                  await element.setStyles(
                    updatedStyles.filter((s): s is Style => s !== null),
                  );
                  pageUpdatedCount++;
                  logs = [...logs, `Updated standalone element ${element.id}`];
                }
              } catch (err) {
                console.warn(
                  `Error processing standalone element ${element.id}:`,
                  err,
                );
              }
            }),
          );

          if (pageUpdatedCount > 0) {
            totalElementsUpdated += pageUpdatedCount;
            logs = [
              ...logs,
              `Updated ${pageUpdatedCount} elements on ${pageName}`,
            ];
          }
        } catch (err) {
          console.error(`Failed to process page ${pageName}:`, err);
          logs = [...logs, `Error on page ${pageName}: ${err}`];
        }
      }

      // 5. Try to remove old styles
      logs = [...logs, `Attempting to remove old styles...`];
      for (const oldStyle of targetStyles) {
        try {
          if (!oldStyle) continue;
          await webflow.removeStyle(oldStyle);
          logs = [...logs, `Removed old style "${await oldStyle.getName()}"`];
        } catch (err) {
          logs = [...logs, `Could not remove style - it may still be in use`];
        }
      }

      logs = [...logs, `Completed! Updated ${totalElementsUpdated} elements`];

      webflow.notify({
        type: "Success",
        message: `Successfully merged styles into "${newStyleName}"`,
      });

      await fetchStyles();
      selectedDuplicates = [];
      newStyleName = "";
    } catch (error) {
      console.error("Style operation failed:", error);
      logs = [
        ...logs,
        `Error: ${error instanceof Error ? error.message : "Failed to update style"}`,
      ];
      webflow.notify({
        type: "Error",
        message:
          error instanceof Error ? error.message : "Failed to update style",
      });
    } finally {
      isProcessing = false;
      currentOperation = "";
      shouldStop = false;

      // Return to original page
      try {
        await webflow.switchPage(originalPage);
      } catch (err) {
        console.warn("Failed to return to original page:", err);
      }
    }
  }

  // Helper function to get template type from page
  async function getPageTemplateType(page: Page) {
    try {
      const pageType = await (page as any).getType();
      if (pageType === "CollectionPage") {
        const collection = await (page as any).getCollectionId();
        if (collection) {
          // Get the template page for this collection
          const templatePages = await (collection as any).getTemplatePages();
          if (templatePages && templatePages.length > 0) {
            // Return the template page ID as our identifier
            return `template-${templatePages[0].id}`;
          }
        }
      }
      return null;
    } catch (err) {
      console.warn("Failed to get page template type:", err);
      return null;
    }
  }

  // Add helper function to check if element is part of a CMS collection
  async function isCollectionItem(element: any): Promise<{
    isCmsItem: boolean;
    listId?: string;
    collectionName?: string;
  }> {
    try {
      // Check for Webflow collection elements
      const collectionTypes = [
        "CollectionList",
        "DynamoList",
        "DynamoWrapper",
        "DynamoItem",
      ];

      // Check if the element itself is a collection element
      if (collectionTypes.includes(element.type)) {
        try {
          const collection = await element.getCollection();
          const collectionName = await collection?.getName();
          return {
            isCmsItem: true,
            listId: element.id,
            collectionName: collectionName || `${element.type} Collection`,
          };
        } catch (err) {
          // If we can't get collection name, use the element type
          console.log(`Collection element found: ${element.type}`);
          return { isCmsItem: true, listId: element.id };
        }
      }

      // Then check its parents
      let current = element;
      while (current.parent) {
        current = current.parent;
        if (collectionTypes.includes(current.type)) {
          // Get the collection info
          try {
            const collection = await current.getCollection();
            const collectionName = await collection?.getName();
            return {
              isCmsItem: true,
              listId: current.id,
              collectionName: collectionName || `${current.type} Collection`,
            };
          } catch (err) {
            console.log(`Parent collection element found: ${current.type}`);
            return { isCmsItem: true, listId: current.id };
          }
        }
      }
      return { isCmsItem: false };
    } catch (err) {
      console.warn("Failed to check collection status:", err);
      return { isCmsItem: false };
    }
  }

  // Add helper function to process elements efficiently
  async function processElementsEfficiently(elements: any[]) {
    const processedElements = new Set<string>();
    const elementsByType = new Map<string, any[]>();
    const processedElementIds = new Set<string>();

    // First, categorize elements by their type
    for (const element of elements) {
      // Skip elements without required methods
      if (!element || typeof element !== "object") continue;

      if (!elementsByType.has(element.type)) {
        elementsByType.set(element.type, []);
      }
      const typeArray = elementsByType.get(element.type) || [];
      typeArray.push(element);
      elementsByType.set(element.type, typeArray);
    }

    // Process DynamoList and DynamoWrapper first to identify collection structures
    const collectionElements = elementsByType.get("DynamoList") || [];
    collectionElements.push(...(elementsByType.get("DynamoWrapper") || []));

    for (const element of collectionElements) {
      try {
        if (element && typeof element.getCollection === "function") {
          const collection = await element.getCollection();
          if (!collection) continue;

          processedElements.add(String(element.id));

          // Only try to get children if the method exists
          if (typeof element.getChildren === "function") {
            const childElements = await element.getChildren();
            for (const child of childElements || []) {
              if (child && child.id) {
                const elementId = String(child.id);
                if (!processedElementIds.has(elementId)) {
                  processedElementIds.add(elementId);
                }
              }
            }
          }
        }
      } catch (err) {
        console.warn(
          `Failed to process collection element of type ${element?.type}:`,
          err,
        );
      }
    }

    return processedElements.size;
  }

  // New helper function: updateElementsUsingStyle
  // This function searches through all pages & elements,
  // replacing any occurrence of the old style (identified by name)
  // with the new style.
  async function updateElementsUsingStyle(
    oldStyleName: string,
    newStyle: Style,
  ) {
    let totalUpdated = 0;
    const processedElementIds = new Set<string>();
    currentOperation = `Updating style: ${oldStyleName}`;
    logs = [...logs, `🔄 Starting update for style "${oldStyleName}"`];

    try {
      // 1. First get all styles and find our target style
      const allStyles = await webflow.getAllStyles();
      logs = [...logs, `📊 Found ${allStyles.length} total styles in project`];

      // Get names for all styles at once
      const styleNames = await Promise.all(
        allStyles.map(async (style) => ({
          style,
          name: await style.getName(),
        })),
      );

      const targetStyle = styleNames.find(
        ({ name }) => name === oldStyleName,
      )?.style;

      if (!targetStyle) {
        logs = [...logs, `⚠️ Style "${oldStyleName}" not found in project`];
        return;
      }
      logs = [...logs, `✅ Found target style: "${oldStyleName}"`];

      // 2. Get all pages that need to be checked - UPDATED
      const pagesAndFolders = await webflow.getAllPagesAndFolders();
      const pages = pagesAndFolders.filter(
        (item: WebflowItem): item is Page =>
          item &&
          typeof item === "object" &&
          "type" in item &&
          item.type === "Page" &&
          "getName" in item &&
          typeof item.getName === "function",
      );

      totalCount = pages.length;
      processedCount = 0;
      logs = [...logs, `🔍 Scanning ${pages.length} pages for style usage`];

      // 3. For each page, get elements and check only those with styles
      for (const page of pages) {
        processedCount++;
        let pageName;
        try {
          pageName = await page.getName();
        } catch (err) {
          pageName = String(page.id);
        }
        logs = [...logs, `📄 Checking page: ${pageName}`];

        try {
          await webflow.switchPage(page as any);
          const elements = await webflow.getAllElements();
          const elementsWithStyles = elements.filter((e) => e?.styles);

          console.log(
            `\n📄 Page: ${pageName}\n` +
              `Total elements: ${elements.length}\n` +
              `Elements with styles: ${elementsWithStyles.length}`,
          );

          // First, quickly check which elements might have our style
          const potentialElements = [];
          for (const element of elementsWithStyles) {
            console.log(
              `\n🔍 Checking element:`,
              `\nID: ${element.id}`,
              `\nType: ${element.type}`,
              `\nTag: ${(element as any).tagName || "N/A"}`,
            );

            try {
              const elementStyles = await (element as any).getStyles();
              console.log(
                `Styles found: ${elementStyles?.length || 0}`,
                elementStyles?.map((s: Style) => s.id),
              );

              if (
                Array.isArray(elementStyles) &&
                elementStyles.some((s) => s.id === targetStyle.id)
              ) {
                console.log(`✅ Found target style on this element!`);
                potentialElements.push(element);
              } else {
                console.log(`❌ Target style not found on this element`);
              }
            } catch (err) {
              // Skip elements that fail style check
              console.log(`⚠️ Failed to check element styles:`, err);
              continue;
            }
          }

          if (potentialElements.length > 0) {
            console.log(
              `\n✨ Summary for ${pageName}:`,
              `\nFound ${potentialElements.length} elements with target style`,
            );
            logs = [
              ...logs,
              `🎯 Found ${potentialElements.length} potential elements on ${pageName}`,
            ];
          } else {
            console.log(
              `\n📝 No elements found with target style on ${pageName}`,
            );
          }

          // 4. Process elements with styles
          for (const element of potentialElements) {
            const elementId = String(element.id);

            // Skip if we've already processed this element
            if (processedElementIds.has(elementId)) {
              continue;
            }

            try {
              const styles = await (element as any).getStyles();
              if (Array.isArray(styles)) {
                // Check if this element uses our target style
                const hasTargetStyle = styles.some(
                  (s) => s.id === targetStyle.id,
                );
                if (hasTargetStyle) {
                  logs = [...logs, `🔄 Updating element ${elementId}`];

                  // Update the element's styles
                  const updatedStyles = styles.map((s) =>
                    s.id === targetStyle.id
                      ? (newStyle as unknown as Style)
                      : s,
                  );
                  await (element as any).setStyles(updatedStyles);
                  totalUpdated++;
                  processedElementIds.add(elementId);
                  logs = [
                    ...logs,
                    `✅ Updated element ${elementId} on ${pageName}`,
                  ];
                }
              }
            } catch (err) {
              logs = [
                ...logs,
                `❌ Failed to update element ${elementId}: ${err}`,
              ];
            }
          }

          if (processedElementIds.size > 0) {
            logs = [
              ...logs,
              `✨ Updated ${processedElementIds.size} elements on ${pageName}`,
            ];
          }
        } catch (err) {
          logs = [...logs, `❌ Error on page ${pageName}: ${err}`];
        }
      }

      // Final summary
      if (totalUpdated > 0) {
        logs = [
          ...logs,
          `🎉 Successfully updated ${totalUpdated} elements with style "${oldStyleName}"`,
        ];
        totalElementsUpdated += totalUpdated;
      } else {
        logs = [...logs, `ℹ️ No elements found using style "${oldStyleName}"`];
      }
    } catch (err) {
      logs = [...logs, `❌ Error: ${err}`];
      console.error("Style update failed:", err);
    }

    // Force Svelte to update the UI
    logs = [...logs];
  }

  // Update the getCollectionTemplateId function to be more reliable
  async function getCollectionTemplateId(element: any): Promise<string | null> {
    try {
      if (!element) return null;

      // Check if element is part of a collection
      const collectionTypes = [
        "CollectionList",
        "DynamoList",
        "DynamoWrapper",
        "DynamoItem",
      ];

      // First check if this element is a collection item
      if (element.type === "DynamoItem") {
        let parent = element.parent;
        while (parent) {
          if (
            parent.type === "DynamoList" &&
            typeof parent.getCollection === "function"
          ) {
            try {
              const collection = await parent.getCollection();
              if (collection?.id) {
                return `collection-${collection.id}`;
              }
            } catch (err) {
              console.warn("Failed to get collection from parent:", err);
            }
          }
          parent = parent.parent;
        }
      }

      // Then check if this element is a collection container
      if (
        element.type === "DynamoList" &&
        typeof element.getCollection === "function"
      ) {
        try {
          const collection = await element.getCollection();
          if (collection?.id) {
            return `collection-${collection.id}`;
          }
        } catch (err) {
          console.warn("Failed to get collection from list:", err);
        }
      }

      return null;
    } catch (err) {
      console.warn("Failed to get collection template ID:", err);
      return null;
    }
  }

  // Add this helper function to get breakpoint styles
  async function getElementBreakpointStyles(element: any) {
    // Use predefined breakpoints instead of webflow.getAllBreakpoints()
    const breakpoints = predefinedBreakpoints;
    const breakpointStyles = new Map();

    // For each breakpoint, get the styles
    for (const breakpoint of breakpoints) {
      try {
        const styles = await element.getBreakpointStyles(breakpoint);
        if (styles && styles.length > 0) {
          breakpointStyles.set(breakpoint.id, styles);
        }
      } catch (err) {
        console.warn(
          `Failed to get styles for breakpoint ${breakpoint.id}:`,
          err,
        );
      }
    }

    return breakpointStyles;
  }

  // Helper function: Get properties for each breakpoint for a given style.
  // This function returns a map where keys are breakpoint IDs and values are the corresponding PropertyMap.
  async function getStylePropertiesForBreakpoints(
    style: Style,
  ): Promise<{ [bp: string]: any }> {
    // Use predefined breakpoints instead of webflow.getAllBreakpoints()
    const breakpoints = predefinedBreakpoints;
    const bpProperties: { [bp: string]: any } = {};
    for (const bp of breakpoints) {
      try {
        // Retrieve the properties for a specific breakpoint.
        // Here we use the 'noPseudo' pseudo state (adjust if needed)
        const properties = await (style as any).getProperties({
          breakpoint: bp.id,
          pseudo: "noPseudo",
        });
        bpProperties[bp.id] = properties;
      } catch (err) {
        console.warn(
          `Error retrieving properties for breakpoint ${bp.id}:`,
          err,
        );
      }
    }
    return bpProperties;
  }

  // Helper function: Identify differences between default (main) style properties and breakpoint-specific properties for a given style.
  async function identifyBreakpointDifferences(
    style: Style,
  ): Promise<{ defaultProperties: any; differences: { [bp: string]: any } }> {
    let defaultProperties: any;
    try {
      // Get the default (main breakpoint) properties
      defaultProperties = await (style as any).getProperties({
        breakpoint: "main",
        pseudo: "noPseudo",
      });
    } catch (err) {
      console.warn("Failed to get default properties for style:", err);
      defaultProperties = {};
    }

    const differences: { [bp: string]: any } = {};
    // Reuse our existing helper to get properties for each breakpoint.
    const bpProperties = await getStylePropertiesForBreakpoints(style);

    // Remove "main" breakpoint from comparison (as that's our default)
    delete bpProperties["main"];

    // Compare each breakpoint's properties to the default properties.
    for (const [bp, props] of Object.entries(bpProperties)) {
      // Simple comparison using JSON stringification.
      if (JSON.stringify(props) !== JSON.stringify(defaultProperties)) {
        differences[bp] = props;
      }
    }

    return { defaultProperties, differences };
  }

  // Type to store breakpoint differences for each style
  type BreakpointDiff = {
    styleName: string;
    defaultProperties: any;
    differences: { [bp: string]: any };
  };

  // Reactive mapping to track which style's differences are expanded (dropdown open)
  let expandedDiffs: { [styleName: string]: boolean } = {};

  // Toggle the expanded state for a given style name
  function toggleDiffVisibility(styleName: string): void {
    expandedDiffs[styleName] = !expandedDiffs[styleName];
  }

  function getDiffForStyle(styleName: string): {
    differences: { [bp: string]: any };
  } {
    const styleData = stylesWithProperties.find((s) => s.name === styleName);
    if (!styleData) return { differences: {} };
    const base = styleData.properties; // This is bpProps["main"]
    if (!base) return { differences: {} };
    const differences: { [bp: string]: any } = {};
    for (const bp in styleData.breakpointProperties) {
      if (bp === "main") continue;
      if (
        JSON.stringify(styleData.breakpointProperties[bp]) !==
        JSON.stringify(base)
      ) {
        differences[bp] = styleData.breakpointProperties[bp];
      }
    }
    return { differences };
  }

  // Helper function to check if an object is empty
  function isEmptyObject(obj: any): boolean {
    return obj && Object.keys(obj).length === 0;
  }

  onMount(async () => {
    await webflow.setExtensionSize("large");
    await fetchStyles();
    // Automatically select the first duplicate group if any exist
    if (duplicates && duplicates.length > 0) {
      selectedGroup = 0;
    }
  });
</script>

<div class="extension-container">
  <!-- Toggle log panel button -->
  <button
    type="button"
    on:click={() => (logPanelVisible = !logPanelVisible)}
    style="margin-bottom: 10px; padding: 8px; border: 1px solid #333; background: transparent; color: white; border-radius: 4px; cursor: pointer;"
  >
    {logPanelVisible ? "Hide Log Panel" : "Show Log Panel"}
  </button>

  <!-- Live Status Panel -->
  {#if isProcessing}
    <div class="live-status-panel">
      <div class="status-header">
        <div class="status-info">
          <div class="status-badge"></div>
          <span>{currentOperation || "Process Complete"}</span>
          {#if isProcessing}
            <span class="status-count"
              >({processedCount}/{totalCount} pages)</span
            >
          {/if}
        </div>
        {#if isProcessing}
          <button
            class="stop-button"
            on:click={() => {
              shouldStop = true;
              logs = [...logs, "Stopping process..."];
            }}
          >
            Stop Process
          </button>
        {/if}
      </div>

      <div class="progress-container">
        <div
          class="progress-bar"
          style="width: {isProcessing
            ? (processedCount / totalCount) * 100
            : 100}%"
        ></div>
      </div>

      <div class="recent-logs">
        {#each logs.slice(-5) as log}
          <div class="log-entry">{log}</div>
        {/each}
      </div>
    </div>
  {:else}
    <div class="status-info">
      {progressInfo}
    </div>
  {/if}

  <!-- Log panel (shown when logPanelVisible is true) -->
  {#if logPanelVisible}
    <div
      class="log-panel"
      style="max-height: 200px; overflow-y: auto; background: #121212; border: 1px solid #333; padding: 8px; margin-bottom: 10px; color: #ccc; font-family: monospace; font-size: 0.85em; border-radius: 4px;"
    >
      {#each logs as logEntry}
        <div>{logEntry}</div>
      {/each}
    </div>
  {/if}

  {#if loading}
    <div class="loading">Loading...</div>
  {:else}
    <div class="search-container">
      <input
        type="text"
        placeholder="Search by style name or property..."
        bind:value={searchTerm}
        class="search-input"
      />
    </div>
    <div class="split-layout">
      <div class="groups-list">
        {#each filteredDuplicates as group, index}
          <button
            class="group-selector"
            class:active={selectedGroup === index}
            on:click={() => {
              selectedGroup = index;
              selectedDuplicates = [];
              newStyleName = "";
            }}
          >
            Duplicates group ({index + 1}) - {group.length} items
          </button>
        {/each}
      </div>
      <div class="preview-panel">
        {#if selectedGroup !== null && selectedGroup < filteredDuplicates.length}
          <div class="group-preview">
            <h3>Style Properties Group ({selectedGroup + 1})</h3>
            <pre class="properties-preview">{filteredPropertiesByGroup[
                selectedGroup
              ]}</pre>
            <ul>
              {#each filteredDuplicates[selectedGroup] as name}
                <div class="merge-item">
                  <label>
                    <input
                      type="checkbox"
                      checked={selectedDuplicates.includes(name)}
                      on:change={(e) => {
                        const checked = (e.target as HTMLInputElement).checked;
                        selectedDuplicates = checked
                          ? [...selectedDuplicates, name]
                          : selectedDuplicates.filter((n) => n !== name);
                      }}
                    />
                    <span class="style-token">{name}</span>
                  </label>
                  <div class="button-group">
                    <button
                      type="button"
                      class="style-button"
                      on:click={() => selectStyleInWebflow(name)}
                    >
                      <!-- Copy SVG icon from asset -->
                      <svg
                        width="14"
                        height="14"
                        viewBox="0 0 24 24"
                        fill="none"
                        xmlns="http://www.w3.org/2000/svg"
                      >
                        <path
                          d="M16 8V5.2C16 4.0799 16 3.51984 15.782 3.09202C15.5903 2.71569 15.2843 2.40973 14.908 2.21799C14.4802 2 13.9201 2 12.8 2H5.2C4.0799 2 3.51984 2 3.09202 2.21799C2.71569 2.40973 2.40973 2.71569 2.21799 3.09202C2 3.51984 2 4.0799 2 5.2V12.8C2 13.9201 2 14.4802 2.21799 14.908C2.40973 15.2843 2.71569 15.5903 3.09202 15.782C3.51984 16 4.0799 16 5.2 16H8M11.2 22H18.8C19.9201 22 20.4802 22 20.908 21.782C21.2843 21.5903 21.5903 21.2843 21.782 20.908C22 20.4802 22 19.9201 22 18.8V11.2C22 10.0799 22 9.51984 21.782 9.09202C21.5903 8.71569 21.2843 8.40973 20.908 8.21799C20.4802 8 19.9201 8 18.8 8H11.2C10.0799 8 9.51984 8 9.09202 8.21799C8.71569 8.40973 8.40973 8.71569 8.21799 9.09202C8 9.51984 8 10.0799 8 11.2V18.8C8 19.9201 8 20.4802 8.21799 20.908C8.40973 21.2843 8.71569 21.5903 9.09202 21.782C9.51984 22 10.0799 22 11.2 22Z"
                          stroke="#ffffff"
                          stroke-width="2"
                          stroke-linecap="round"
                          stroke-linejoin="round"
                        />
                      </svg>
                    </button>
                    <button
                      type="button"
                      class="style-button remove"
                      on:click={() => removeStyle(name)}
                    >
                      <!-- Trash (Remove) SVG icon from asset -->
                      <svg
                        width="14"
                        height="14"
                        viewBox="0 0 20 22"
                        fill="none"
                        xmlns="http://www.w3.org/2000/svg"
                      >
                        <path
                          d="M14 5V4.2C14 3.0799 14 2.51984 13.782 2.09202C13.5903 1.71569 13.2843 1.40973 12.908 1.21799C12.4802 1 11.9201 1 10.8 1H9.2C8.07989 1 7.51984 1 7.09202 1.21799C6.71569 1.40973 6.40973 1.71569 6.21799 2.09202C6 2.51984 6 3.0799 6 4.2V5M8 10.5V15.5M12 10.5V15.5M1 5H19M17 5V16.2C17 17.8802 17 18.7202 16.673 19.362C16.3854 19.9265 15.9265 20.3854 15.362 20.673C14.7202 21 13.8802 21 12.2 21H7.8C6.11984 21 5.27976 21 4.63803 20.673C4.07354 20.3854 3.6146 19.9265 3.32698 19.362C3 18.7202 3 17.8802 3 16.2V5"
                          stroke="#ff4444"
                          fill="none"
                          stroke-width="2"
                          stroke-linecap="round"
                          stroke-linejoin="round"
                        />
                      </svg>
                    </button>
                  </div>
                </div>
                <div class="inline-diff">
                  <span
                    class="toggle-diff"
                    role="button"
                    tabindex="0"
                    on:click={() => toggleDiffVisibility(name)}
                    on:keyup={(e) => {
                      if (e.key === "Enter" || e.key === " ")
                        toggleDiffVisibility(name);
                    }}
                  >
                    {expandedDiffs[name]
                      ? " Hide Differences"
                      : " Show Differences"}
                  </span>
                  {#if expandedDiffs[name]}
                    {#if getDiffForStyle(name)}
                      {#if Object.keys(getDiffForStyle(name).differences).length > 0}
                        <ul>
                          {#each Object.entries(getDiffForStyle(name).differences) as [bp, diff]}
                            {#if !isEmptyObject(diff)}
                              <li>
                                <strong
                                  >{breakpointDisplayNames[bp] || bp}:</strong
                                >
                                <pre>{JSON.stringify(diff, null, 2)}</pre>
                              </li>
                            {/if}
                          {/each}
                        </ul>
                      {:else}
                        <p>No breakpoint differences.</p>
                      {/if}
                    {/if}
                  {/if}
                </div>
              {/each}
            </ul>
            <div class="merge-section">
              <input
                type="text"
                placeholder="Enter new style name"
                bind:value={newStyleName}
              />
              <button
                on:click={handleCreateStyle}
                disabled={!newStyleName || !selectedDuplicates[0]}
              >
                Create & merge new style
              </button>
            </div>
          </div>
        {:else}
          <div class="empty-state">Select a group to view details</div>
        {/if}
      </div>
    </div>
  {/if}
</div>

<style>
  .extension-container {
    margin: 10px;
    padding: 10px;
    background: black;
    min-height: calc(100vh - 20px);
    border-radius: 4px;
  }
  .split-layout {
    display: grid;
    grid-template-columns: 300px 1fr;
    gap: 20px;
    height: calc(100vh - 120px);
  }
  .groups-list {
    border-right: 1px solid #333;
    padding-right: 16px;
    overflow-y: auto;
  }
  .preview-panel {
    overflow-y: auto;
    padding: 0 16px;
  }
  .group-selector {
    width: 100%;
    text-align: left;
    padding: 8px;
    margin-bottom: 8px;
    border-radius: 4px;
    cursor: pointer;
    background: transparent;
    border: 1px solid #333;
  }
  .group-selector.active {
    background: #333;
  }
  .empty-state {
    text-align: center;
    padding: 40px;
    color: #666;
  }
  .merge-item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 0.5rem;
  }
  .button-group {
    display: flex;
    gap: 10px;
  }
  .merge-item label {
    display: flex;
    align-items: center;
    gap: 8px;
    color: white;
  }
  .style-button {
    padding: 0;
    background: transparent;
    border: none;
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    cursor: pointer;
  }
  .style-button:hover {
    background: rgba(255, 255, 255, 0.1);
    border-radius: 4px;
  }
  .properties-preview {
    font-size: 0.8em;
    font-family: "Monaco", "Menlo", "Ubuntu Mono", "Consolas", monospace;
    color: #d4d4d4;
    background: #1e1e1e;
    padding: 16px;
    margin: 8px 0;
    border-radius: 6px;
    max-height: 200px;
    overflow: auto;
    white-space: pre-wrap;
    line-height: 1.5;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15);
    border: 1px solid #333;
  }
  .search-container {
    margin-bottom: 20px;
    margin-top: 16px;
    padding: 0 16px;
  }
  .search-input {
    width: 100%;
    padding: 8px;
    background: #1e1e1e;
    border: 1px solid #333;
    border-radius: 4px;
    color: white;
  }
  .merge-section {
    margin-top: 20px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }
  .merge-section input {
    padding: 8px;
    background: #1e1e1e;
    border: 1px solid #333;
    border-radius: 4px;
    color: white;
  }
  .merge-section button {
    padding: 8px 16px;
    border-radius: 4px;
    border: 1px solid #333;
    background: transparent;
    color: white;
    cursor: pointer;
  }
  .merge-section button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  .merge-section button:hover:not(:disabled) {
    background: #333;
  }
  .live-status-panel {
    background: #1a1a1a;
    border: 1px solid #333;
    border-radius: 8px;
    padding: 16px;
    margin-bottom: 20px;
  }
  .status-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 12px;
  }
  .status-info {
    display: flex;
    align-items: center;
    gap: 8px;
    color: #fff;
  }
  .status-badge {
    width: 8px;
    height: 8px;
    background: #4caf50;
    border-radius: 50%;
    animation: pulse 1s infinite;
  }
  .status-count {
    color: #666;
  }
  .stop-button {
    background: #ff4444;
    color: white;
    border: none;
    padding: 6px 12px;
    border-radius: 4px;
    cursor: pointer;
    font-size: 12px;
  }
  .progress-container {
    background: #121212;
    border-radius: 4px;
    height: 4px;
    margin: 12px 0;
    overflow: hidden;
  }
  .progress-bar {
    height: 100%;
    background: #4caf50;
    transition: width 0.3s ease;
  }
  .recent-logs {
    margin-top: 12px;
    font-family: monospace;
    font-size: 12px;
  }
  .log-entry {
    padding: 4px 8px;
    color: #ccc;
    border-left: 2px solid #4caf50;
    margin: 4px 0;
  }
  @keyframes pulse {
    0% {
      opacity: 1;
    }
    50% {
      opacity: 0.5;
    }
    100% {
      opacity: 1;
    }
  }
  /* New style for style tokens in the duplicate list to mimic Webflow's UI */
  .style-token {
    cursor: inherit;
    user-select: none;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    box-sizing: border-box;
    min-width: 0;
    max-width: 100%;
    padding-left: 6px;
    padding-right: 6px;
    border-radius: 2px;
    height: 24px;
    background: #006acc;
    font-size: 0.95rem;
    text-shadow: none;
    box-shadow: rgba(0, 0, 0, 0.15) 0px 0px 0px 1px;
    overflow: hidden;
    color: white;
  }
  .toggle-diff {
    margin: 2px;
    color: white;
    cursor: pointer;
    font-size: 0.85rem;
  }
  /* Ensure the dropdown block is displayed below the merge item */
  .inline-diff {
    display: block;
    margin-top: 0.5rem;
    margin-bottom: 1rem;
    margin-left: 2rem; /* Optional indent for clarity */
    font-size: 0.85rem;
    color: #ccc;
  }
  .inline-diff ul {
    margin-top: 0.5rem;
  }
</style>
