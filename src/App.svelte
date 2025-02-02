<script lang="ts">
  /// <reference types="@webflow/designer-extension-typings" />
  import { onMount, tick } from "svelte";

  let stylesWithProperties: Array<{
    name: string;
    properties: any;
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

  const fetchStyles = async () => {
    loading = true;
    try {
      const styles = await webflow.getAllStyles();

      stylesWithProperties = await Promise.all(
        styles.map(async (style) => ({
          name: await style.getName(),
          properties: await style.getProperties(),
        })),
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
    styles: Array<{ name: string; properties: any }>,
  ) => {
    const duplicates: string[][] = [];
    const propertiesByGroup: string[] = [];
    const processed = new Set<number>();

    for (let i = 0; i < styles.length; i++) {
      const { name: name1, properties: properties1 } = styles[i];
      if (processed.has(i) || Object.keys(properties1).length === 0) continue;
      const duplicateGroup = [name1];

      for (let j = i + 1; j < styles.length; j++) {
        if (processed.has(j)) continue;
        const { name: name2, properties: properties2 } = styles[j];
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
      const style = await webflow.getStyleByName(styleName);
      if (!style) {
        throw new Error("Style not found");
      }

      // First try to remove all properties
      await style.removeAllProperties();

      // Then try to remove the style itself
      await webflow.removeStyle(style);

      webflow.notify({
        type: "Success",
        message: `Successfully removed style "${styleName}"`,
      });

      await fetchStyles();
    } catch (error) {
      console.error(`Error removing style: ${styleName}`, error);
      webflow.notify({
        type: "Error",
        message:
          "Could not remove style. Make sure it's not being used by any elements.",
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

  async function handleCreateStyle() {
    try {
      const existingStyle = await webflow.getStyleByName(newStyleName);
      if (existingStyle) {
        throw new Error(
          "Style with this name already exists. Please choose a different name.",
        );
      }

      // Create new style
      const newStyle = await webflow.createStyle(newStyleName);

      // Get properties from first selected style
      const sourceStyle = await webflow.getStyleByName(selectedDuplicates[0]);
      const properties = await sourceStyle?.getProperties();

      if (properties) {
        // Validate and clean overflow properties
        const cleanedProperties = { ...properties } as WebflowStyleProperties;

        // Handle specific Webflow style properties
        const propertyValidations: Record<string, readonly string[]> = {
          "word-wrap": ["break-word"],
          "grid-row-gap": ["normal"],
          "grid-column-gap": ["normal"],
          "grid-gap": ["normal"],
          "background-clip": ["border-box", "padding-box", "content-box"],
          "background-origin": ["border-box", "padding-box", "content-box"],
        };

        // Property name mappings (short to long form)
        const propertyMappings: Record<string, string> = {
          "row-gap": "grid-row-gap",
          "column-gap": "grid-column-gap",
          gap: "grid-gap",
          "overflow-wrap": "word-wrap",
        };

        // Clean up properties based on Webflow's supported values
        Object.entries(propertyMappings).forEach(([shortForm, longForm]) => {
          if (shortForm in cleanedProperties) {
            const value = cleanedProperties[shortForm];
            if (value && (typeof value === "string" || isStyleValue(value))) {
              cleanedProperties[longForm] = value;
            }
            delete cleanedProperties[shortForm];
          }
        });

        // Then validate the values
        for (const [prop, validValues] of Object.entries(propertyValidations)) {
          const value = cleanedProperties[prop];
          if (value) {
            const stringValue =
              typeof value === "string"
                ? value
                : isStyleValue(value)
                  ? value.value
                  : null;

            if (stringValue && !validValues.includes(stringValue)) {
              console.warn(`Invalid ${prop} value removed: ${stringValue}`);
              delete cleanedProperties[prop];
            }
          }
        }

        // Handle aspect ratio alternative
        if ("aspect-ratio" in cleanedProperties) {
          console.warn(
            "Aspect ratio not supported, using padding-bottom technique",
          );
          const ratioValue = cleanedProperties["aspect-ratio"];
          const ratio =
            typeof ratioValue === "string"
              ? ratioValue
              : ratioValue && isStyleValue(ratioValue)
                ? ratioValue.value
                : null;
          delete cleanedProperties["aspect-ratio"];

          if (ratio) {
            try {
              const [width, height] = ratio.split("/").map(Number);
              if (!isNaN(width) && !isNaN(height)) {
                const paddingValue: StyleValue = {
                  value: `${(height / width) * 100}%`,
                };
                cleanedProperties["padding-top"] = paddingValue;
                cleanedProperties["height"] = { value: "0" } as StyleValue;
                cleanedProperties["display"] = { value: "block" } as StyleValue;
              }
            } catch (err) {
              console.warn("Failed to parse aspect ratio:", ratio);
            }
          }
        }

        // Set properties on new style
        await newStyle.setProperties(cleanedProperties);

        // Get all pages and folders, then filter for just pages
        const pagesAndFolders = await webflow.getAllPagesAndFolders();
        const pages = pagesAndFolders?.filter((item) => item.type === "Page");

        if (!pages || pages.length === 0) {
          throw new Error("No pages found in the project");
        }

        let updatedCount = 0;
        let processedPages = 0;
        const originalPage = await webflow.getCurrentPage();

        // Track which pages have elements using our styles
        const pageMap = new Map<
          string,
          { pageName: string; elementCount: number }
        >();

        // Initialize progress info and log start
        progressInfo = "Starting page processing...";
        logs.push(progressInfo);

        try {
          // Process each page
          for (const page of pages) {
            try {
              // Switch to the page context
              await webflow.switchPage(page);
              processedPages++;

              // Get page name
              const pageName = await page.getName();

              // Reduced wait time for page load; adjust as needed
              await new Promise((resolve) => setTimeout(resolve, 300));

              // Get elements for current page
              const elements = await webflow.getAllElements();
              let pageElementCount = 0;

              for (const element of elements) {
                if (element.styles) {
                  try {
                    const elementStyles = await element.getStyles();
                    if (!elementStyles) continue;

                    // Get names of all styles on this element
                    const elementStyleNames = await Promise.all(
                      elementStyles.map(async (style: Style | null) => {
                        if (!style) return null;
                        try {
                          return await style.getName();
                        } catch {
                          return null;
                        }
                      }),
                    );

                    // Check if element has any of the selected styles
                    const hasSelectedStyle = elementStyleNames.some(
                      (name: string | null): name is string =>
                        typeof name === "string" &&
                        selectedDuplicates.includes(name),
                    );

                    if (hasSelectedStyle) {
                      // Keep only the styles that aren't being replaced
                      const keepStyles = elementStyles.filter(
                        (style: Style | null, index: number) =>
                          style &&
                          elementStyleNames[index] &&
                          !selectedDuplicates.includes(
                            elementStyleNames[index] as string,
                          ),
                      );

                      // Add the new style to the kept styles
                      if (newStyle) {
                        const validStyles = keepStyles.filter(
                          (style: Style | null): style is Style =>
                            style !== null,
                        );
                        await element.setStyles([...validStyles, newStyle]);
                      }
                      updatedCount++;
                      pageElementCount++;
                    }
                  } catch (err) {
                    console.warn("Failed to process element styles:", err);
                  }
                }
              }

              // If we found elements on this page, add it to our map
              if (pageElementCount > 0) {
                pageMap.set(page.id, {
                  pageName: pageName,
                  elementCount: pageElementCount,
                });
              }

              // Update progress info and add to log
              progressInfo = `Processed page "${pageName}": ${pageElementCount} element(s) updated. (${processedPages}/${pages.length} pages processed)`;
              logs.push(progressInfo);

              // Reduced delay between page switches
              await new Promise((resolve) => setTimeout(resolve, 300));
            } catch (pageErr) {
              const errMsg = `Failed to process page ${await page.getName()}: ${pageErr}`;
              console.warn(errMsg);
              logs.push(errMsg);
            }
          }

          // Switch back to original page
          await webflow.switchPage(originalPage);
          progressInfo = "Page processing complete.";
          logs.push(progressInfo);
        } catch (err) {
          console.warn("Failed to process pages:", err);
          // Make sure we return to the original page even if there's an error
          try {
            await webflow.switchPage(originalPage);
          } catch (switchErr) {
            console.warn("Failed to switch back to original page:", switchErr);
          }
        }

        // Create a summary of changes
        const pagesSummary = Array.from(pageMap.entries())
          .map(
            ([_, info]) =>
              `\n- ${info.pageName}: ${info.elementCount} elements`,
          )
          .join("");

        const summaryMessage = `Created new style "${newStyleName}".
Total elements updated: ${updatedCount}.
Processed ${processedPages} pages.
Changes made on pages:
${pagesSummary}
${processedPages < pages.length ? `\nWarning: Only processed ${processedPages} out of ${pages.length} pages.` : ""}
${
  Object.keys(properties).some((key) => !(key in cleanedProperties))
    ? "\nSome properties were adjusted to match Webflow constraints. Check console for details."
    : ""
}`;

        // Add the summary to the log panel
        logs.push(summaryMessage);

        webflow.notify({
          type: "Success",
          message: summaryMessage,
        });

        // Remove duplicate styles that are no longer used (keep only the new merged style)
        for (const duplicateStyle of selectedDuplicates) {
          if (duplicateStyle !== newStyleName) {
            try {
              await removeStyle(duplicateStyle);
              logs.push(`Removed unused style "${duplicateStyle}".`);
            } catch (err) {
              logs.push(`Failed to remove style "${duplicateStyle}": ${err}`);
            }
          }
        }

        await fetchStyles();
        selectedDuplicates = [];
        newStyleName = "";
      }
    } catch (error) {
      console.error("Style operation failed:", error);
      webflow.notify({
        type: "Error",
        message:
          error instanceof Error ? error.message : "Failed to update style",
      });
    }
  }

  onMount(async () => {
    await webflow.setExtensionSize("large");
    await fetchStyles();
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

  <!-- Progress display -->
  <div
    class="progress-info"
    style="margin-bottom: 10px; padding: 8px; background: #222; color: #fff; border-radius: 4px;"
  >
    {progressInfo}
  </div>

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
                    {name}
                  </label>
                  <div class="button-group">
                    <button
                      type="button"
                      class="style-button"
                      on:click={() => selectStyleInWebflow(name)}
                    >
                      Copy
                    </button>
                    <button
                      type="button"
                      class="style-button remove"
                      on:click={() => removeStyle(name)}
                    >
                      Remove
                    </button>
                  </div>
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
                Create New Style
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
    gap: 8px;
  }
  .merge-item label {
    display: flex;
    align-items: center;
    gap: 8px;
    color: white;
  }
  .style-button {
    padding: 8px;
    background: transparent;
    border: 1px solid #333;
    border-radius: 4px;
    color: white;
    cursor: pointer;
  }
  .style-button.remove {
    border-color: #ff4444;
    color: #ff4444;
  }
  .style-button.remove:hover {
    background: rgba(255, 68, 68, 0.1);
  }
  .style-button:hover {
    background: #333;
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
</style>
