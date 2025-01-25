<script lang="ts">
  /// <reference types="@webflow/designer-extension-typings" />
  import { onMount } from "svelte";

  let stylesWithProperties: Array<{
    name: string;
    properties: any;
    isCombo: boolean;
  }> = [];
  let duplicates: Array<string[]> = [];
  let propertiesByGroup: Array<string> = [];
  let loading = false;
  let searchTerm = "";
  let selectedGroup: number | null = null;

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
          isCombo: (await style.getName()).includes(" "),
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
    styles: Array<{ name: string; properties: any; isCombo: boolean }>,
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
        // Create temporary input element
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

  onMount(async () => {
    await webflow.setExtensionSize("large");
    await fetchStyles();
  });
</script>

<div class="extension-container">
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
            on:click={() => (selectedGroup = index)}
          >
            Duplicates group ({index + 1}) - {group.length} items
          </button>
        {/each}
      </div>
      <div class="preview-panel">
        {#if selectedGroup !== null && selectedGroup < filteredDuplicates.length}
          <div class="group-preview">
            <h3>Duplicates group ({selectedGroup + 1})</h3>
            <pre class="properties-preview">{filteredPropertiesByGroup[
                selectedGroup
              ]}</pre>
            <ul>
              {#each filteredDuplicates[selectedGroup] as name}
                <button
                  type="button"
                  class="style-button"
                  on:click={() => selectStyleInWebflow(name)}
                >
                  {name}
                </button>
              {/each}
            </ul>
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
  .style-button {
    margin-bottom: 0.5rem;
  }
  .style-button:last-child {
    margin-bottom: 0;
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
  }
</style>
