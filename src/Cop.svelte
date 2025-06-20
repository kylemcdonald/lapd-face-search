<script>
  let { cop, distance, index } = $props();

  const DEV = import.meta.env.DEV;

  const errorImage =
    "data:image/svg+xml;base64,PHN2ZyB3aWR0aD0iODAiIGhlaWdodD0iODAiIHZpZXdCb3g9IjAgMCA4MCA4MCIgZmlsbD0ibm9uZSIgeG1sbnM9Imh0dHA6Ly93d3cudzMub3JnLzIwMDAvc3ZnIj4KPHJlY3Qgd2lkdGg9IjgwIiBoZWlnaHQ9IjgwIiBmaWxsPSIjRjNGNEY2Ii8+CjxwYXRoIGQ9Ik0yNCAzMkM0MS42NzMxIDMyIDU2IDM3LjM3MjYgNTYgNDRDNTYgNTAuNjI3NCA0MS42NzMxIDU2IDI0IDU2QzYuMzI2ODggNTYgLTggNTAuNjI3NCAtOCA0Qy04IDM3LjM3MjYgNi4zMjY4OCAzMiAyNCAzMloiIGZpbGw9IiM5QTlCOCIvPgo8L3N2Zz4=";

  let name = $derived.by(() => {
    if (cop.name) {
      return cop.name;
    } else if (cop.filename) {
      // Fallback to extracting from filename
      return cop.filename.split(".")[0];
    } else if (cop.image) {
      // Fallback to extracting from image filename
      return cop.image.split(".")[0];
    } else {
      console.warn(`No name or filename for match ${index}:`, cop);
      return `Face ${index}`;
    }
  });

  let imageSrc = $derived(`/images/${cop.filename}`);

  let url = $derived.by(() => {
    if (cop.serial && cop.name) {
      const nameForUrl = cop.name.toLowerCase().replace(/\s+/g, "-");
      return `https://watchthewatchers.net/lapd/cop/${cop.serial}/${nameForUrl}`;
    } else {
      return null;
    }
  });

  let hasSpaces = $derived(name && name.includes(" "));
</script>

<div class="cop">
  <div class="cop-face">
    {#if url}
      <a href={url} target="_blank">
        <img
          src={imageSrc}
          onerror={() => (imageSrc = errorImage)}
          alt={name}
        />
      </a>
    {:else}
      <img src={imageSrc} onerror={() => (imageSrc = errorImage)} alt={name} />
    {/if}
  </div>
  <div class="cop-info">
    <div class="name">{name}</div>
    <div class="serial">Serial: {cop.serial || "N/A"}</div>
    <div class="link">
      {#if url && hasSpaces}
        <a href={url} target="_blank" class="view-profile-link">View Profile</a>
      {:else}
        <span class="no-profile-link">No Profile</span>
      {/if}
    </div>
  </div>
</div>

<style>
  .cop {
    text-transform: uppercase;
    overflow: hidden;
    text-align: center;
    padding-bottom: 5px;
    display: flex;
    flex-direction: column;
    justify-content: space-between;
  }
  .cop-info {
    padding: 1em;
  }

  img {
    width: 100%;
    display: block;
  }

  .serial {
    font-size: 1.2em;
    color: rgba(255, 255, 255, 0.7);
  }

  .name {
    font-size: 1.4em;
    margin-bottom: 5px;
    font-weight: bold;
  }

  .link {
    margin-top: 15px;
  }

  .view-profile-link {
    font-size: 1.2em;
    background-color: red;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    text-transform: uppercase;
    padding: 0.4em 1em;
    cursor: pointer;
    color: #fff;
    border: 2px solid red;
    transition: all 0.2s ease-in-out;
    text-decoration: none;
  }
  .view-profile-link:hover {
    background-color: transparent;
    color: red;
  }

  .no-profile-link {
    font-size: 1.2em;
    background-color: #666;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    text-transform: uppercase;
    padding: 0.4em 1em;
    color: #999;
    border: 2px solid #666;
    cursor: not-allowed;
  }
</style>
