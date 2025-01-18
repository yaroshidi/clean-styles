<script lang="ts">
  /// <reference types="@webflow/designer-extension-typings" />
  import { onMount } from "svelte";

  let stylesWithProperties: Array<{ name: string; properties: any }> = [];
  let duplicates: Array<string[]> = [];
  let propertiesByGroup: Array<string> = [];
  let loading = false;

  const fetchStyles = async () => {
    loading = true;
    try {
      // Get all style instances from the current page
      const styles = await webflow.getAllStyles();

      // Map styles to name and properties
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
      if (processed.has(i)) continue;
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
      const style = await webflow.getStyleByName(styleName);
      if (style) {
        // You might want to implement selection logic here
        console.log(`Selected style: ${styleName}`);
      }
    } catch (error) {
      console.error(`Error selecting style: ${styleName}`, error);
    }
  };

  onMount(fetchStyles);
</script>

<div>
  {#if loading}
    <div class="loading">Loading...</div>
  {:else}
    <div id="duplicates-container">
      {#each duplicates as group, index}
        <details class="duplicate-group">
          <summary class="group-title">
            Duplicates group ({index + 1}) - {group.length} items
          </summary>
          <details class="properties-details">
            <summary>Show properties</summary>
            <pre class="properties-preview">{propertiesByGroup[index]}</pre>
          </details>
          <ul>
            {#each group as name}
              <button
                type="button"
                class="style-button"
                on:click={() => selectStyleInWebflow(name)}
              >
                {name}
              </button>
            {/each}
          </ul>
        </details>
      {/each}
    </div>
  {/if}
</div>

<style>
  .duplicate-group {
    margin-bottom: 12px;
  }
  .group-title {
    cursor: pointer;
    padding: 8px;
    font-weight: bold;
  }
  ul {
    margin: 8px 0;
    padding-left: 24px;
  }
  .style-button {
    margin-bottom: 0.5rem;
  }
  .style-button:last-child {
    margin-bottom: 0;
  }
  .properties-preview {
    font-size: 0.8em;
    color: #666;
    background: #f5f5f5;
    padding: 8px;
    margin: 4px 0;
    border-radius: 4px;
    max-height: 200px;
    overflow: auto;
    white-space: pre-wrap;
  }
  .properties-details {
    margin: 8px 0;
    padding-left: 16px;
  }
  .properties-details summary {
    cursor: pointer;
    margin-bottom: 8px;
  }
</style>
