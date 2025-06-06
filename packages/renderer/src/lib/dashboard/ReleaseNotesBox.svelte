<script lang="ts">
import { faCircleArrowUp } from '@fortawesome/free-solid-svg-icons';
import { Button, CloseButton, Link } from '@podman-desktop/ui-svelte';
import { onDestroy, onMount } from 'svelte';

import { onDidChangeConfiguration } from '/@/stores/configurationProperties';
import { updateAvailable } from '/@/stores/update-store';
import type { ReleaseNotes } from '/@api/release-notes-info';

import Markdown from '../markdown/Markdown.svelte';

let showBanner = $state(false);
let notesAvailable = $state(false);
let notesURL: string | undefined = $state();
let currentVersion: string | undefined = $state();
let notesInfo: ReleaseNotes | undefined = $state();
let imageError: boolean = $state(false);

function onDidChangeConfigurationCallback(e: Event): void {
  if (!('detail' in e) || !e.detail || typeof e.detail !== 'object') {
    return;
  }
  if ('key' in e.detail && 'value' in e.detail) {
    const detail = e.detail as { key: string; value: string };
    if ('releaseNotesBanner.show' === detail.key) {
      showBanner = detail.value !== currentVersion ? true : false;
    }
  }
}

async function openReleaseNotes(): Promise<void> {
  if (!notesURL) return;
  await window.openExternal(notesURL);
}

async function updatePodmanDesktop(): Promise<void> {
  await window.updatePodmanDesktop();
}

async function getInfoFromNotes(): Promise<void> {
  const releaseNotes = await window.podmanDesktopGetReleaseNotes();
  notesInfo = releaseNotes.notes;
  notesAvailable = notesInfo !== undefined;
  notesURL = releaseNotes.notesURL;
}

async function onClose(): Promise<void> {
  await window.updateConfigurationValue(`releaseNotesBanner.show`, currentVersion);
  showBanner = false;
}

onMount(async () => {
  onDidChangeConfiguration.addEventListener('releaseNotesBanner.show', onDidChangeConfigurationCallback);
  currentVersion = await window.getPodmanDesktopVersion();
  showBanner = (await window.getConfigurationValue(`releaseNotesBanner.show`)) !== currentVersion ? true : false;
  await getInfoFromNotes();
});

function onImageError(): void {
  imageError = true;
}

onDestroy(async () => {
  onDidChangeConfiguration.removeEventListener('releaseNotesBanner.show', onDidChangeConfigurationCallback);
});
</script>

{#if showBanner}
  {#if notesAvailable}
    <div class="flex bg-[var(--pd-content-card-bg)] rounded-md p-5 gap-3 flex-row flex-nowrap h-[200px] items-center">
      {#if notesInfo?.image && !imageError}
        <img
          src={notesInfo.image}
          onerror={onImageError}
          class="max-h-[100%] w-auto max-w-[20%] object-contain rounded-md self-start"
          alt={`Podman Desktop ${currentVersion} release image`} />
      {/if}
      <div class="flex flex-col flex-1 h-100% self-start">
        <div class="flex flex-row items-center justify-between">
          <p class="text-[var(--pd-content-card-header-text)] font-bold text-xl ml-2">
            {notesInfo?.title ?? ''}
          </p>
          <CloseButton onclick={onClose} />
        </div>
        {#if notesInfo?.summary}
          <div
            class="text-[var(--pd-content-card-text)] truncate text-ellipsis overflow-hidden whitespace-pre-line line-clamp-6">
            <Markdown markdown={notesInfo?.summary}></Markdown>
          </div>
        {/if}
        <div class="flex flex-row justify-end items-center gap-3 mt-2">
          <Link on:click={openReleaseNotes}>Learn more</Link>
          <Button on:click={updatePodmanDesktop} hidden={!$updateAvailable} icon={faCircleArrowUp}>Update</Button>
        </div>
      </div>
    </div>
  {:else if notesURL}
    <div class="flex bg-[var(--pd-content-card-bg)] rounded-md p-5 flex-col flex-nowrap h-auto items-center">
      <div class="flex flex-row items-center justify-between w-full">
        <p class="text-[var(--pd-content-card-header-text)] font-bold text-lg w-full items-center">
          Release notes are currently unavailable, please check again later
          {#if notesURL}
            or try this <Link on:click={openReleaseNotes}>link</Link>
          {/if}
        </p>
        <CloseButton onclick={onClose} />
      </div>
    </div>
  {/if}
{/if}
