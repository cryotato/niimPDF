<file path="src/components/PrinterConnector.svelte">
<script lang="ts">
  import { NiimbotCapacitorBleClient, SoundSettingsItemType, Utils, type AvailableTransports } from "@mmote/niimbluelib";
  import {
    printerClient,
    connectedPrinterName,
    connectionState,
    initClient,
    heartbeatData,
    printerInfo,
    printerMeta,
    heartbeatFails,
    automation,
    rfidInfo,
    ribbonRfidInfo,
    refreshRfidInfo,
  } from "$/stores";
  import type { ConnectionType } from "$/types";
  import { tr } from "$/utils/i18n";
  import MdIcon from "$/components/basic/MdIcon.svelte";
  import { Toasts } from "$/utils/toasts";
  import { onMount } from "svelte";
  import { LocalStoragePersistence } from "$/utils/persistence";
  import type { MaterialIcon } from "material-icons";
  import FirmwareUpdater from "$/components/basic/FirmwareUpdater.svelte";

  let connectionType = $state<ConnectionType>("bluetooth");
  let featureSupport = $state<AvailableTransports>({ webBluetooth: false, webSerial: false, capacitorBle: false });

  const onConnectClicked = async (autoScanB1 = false) => {
    initClient(connectionType);
    connectionState.set("connecting");

    try {
      if ($printerClient instanceof NiimbotCapacitorBleClient) {
        if (autoScanB1) {
          try {
            const { BleClient } = await import("@capacitor-community/bluetooth-le");
            await BleClient.initialize();
            let foundId: string | undefined = undefined;

            await BleClient.requestLEScan({}, (result) => {
              if (result.device.name && result.device.name.includes("B1")) {
                foundId = result.device.deviceId;
              }
            });

            // Wait up to 3 seconds for scan
            for (let i = 0; i < 30; i++) {
              await new Promise(r => setTimeout(r, 100));
              if (foundId) break;
            }
            await BleClient.stopLEScan();

            if (foundId) {
              await $printerClient.connect({ deviceId: foundId });
              return;
            } else {
              connectionState.set("disconnected");
              return;
            }
          } catch (e) {
            console.error("Auto-scan error", e);
            connectionState.set("disconnected");
            return;
          }
        }

        if ($automation?.autoConnectDeviceId !== undefined) {
          await $printerClient.connect({ deviceId: $automation.autoConnectDeviceId });
        } else {
          await $printerClient.connect();
        }
      } else {
        await $printerClient.connect();
      }
    } catch (e) {
      connectionState.set("disconnected");
      if (!autoScanB1) Toasts.error(e);
    }
  };

  const onDisconnectClicked = () => {
    $printerClient.disconnect();
  };

  const startHeartbeat = async () => {
    $printerClient.startHeartbeat();
  };

  const stopHeartbeat = async () => {
    $printerClient.stopHeartbeat();
  };

  const soundOn = async () => {
    await $printerClient.abstraction.setSoundEnabled(SoundSettingsItemType.BluetoothConnectionSound, true);
    await $printerClient.abstraction.setSoundEnabled(SoundSettingsItemType.PowerSound, true);
  };

  const soundOff = async () => {
    await $printerClient.abstraction.setSoundEnabled(SoundSettingsItemType.BluetoothConnectionSound, false);
    await $printerClient.abstraction.setSoundEnabled(SoundSettingsItemType.PowerSound, false);
  };

  const fetchInfo = async () => {
    await $printerClient.fetchPrinterInfo();
  };

  const reset = async () => {
    await $printerClient.abstraction.printerReset();
  };

  const switchConnectionType = (c: ConnectionType) => {
    LocalStoragePersistence.saveLastConnectionType(c);
    connectionType = c;
  };

  const batteryIcon = (value: number): MaterialIcon => {
    if (value > 4) {
      value = Math.min(4, Math.max(1, Math.ceil(value / 25)));
    }

    if (value === 4) {
      return "battery_full";
    } else if (value === 3) {
      return "battery_5_bar";
    } else if (value === 2) {
      return "battery_3_bar";
    } else if (value === 1) {
      return "battery_2_bar";
    }
    return "battery_0_bar";
  };

  onMount(() => {
    featureSupport = Utils.getAvailableTransports();

    connectionType = LocalStoragePersistence.loadLastConnectionType() ?? "bluetooth";

    if (!featureSupport.capacitorBle && connectionType === "capacitor-ble") {
      connectionType = "bluetooth";
    }
    if (!featureSupport.webSerial && connectionType === "serial") {
      connectionType = "bluetooth";
    }
    if (!featureSupport.webBluetooth && connectionType === "bluetooth" && featureSupport.capacitorBle) {
      connectionType = "capacitor-ble";
    }

    if (connectionType === "capacitor-ble") {
      onConnectClicked(true);
    } else if ($automation !== undefined && $automation.autoConnect && connectionType === "capacitor-ble") {
      onConnectClicked(false);
    }
  });
</script>

<div class="input-group w-auto input-group-sm flex-nowrap justify-content-end">
  {#if $connectionState === "connected"}
    <button class="btn btn-secondary" data-bs-toggle="dropdown" data-bs-auto-close="outside">
      <MdIcon icon="settings" />
    </button>
    <div class="dropdown-menu p-1">
      {#if $printerInfo}
        <div>
          Printer info:
          <ul>
            {#each Object.entries($printerInfo) as [key, value] (key)}
              <li>{key}: <strong>{value ?? "-"}</strong></li>
            {/each}
          </ul>
        </div>
      {/if}

      {#if $printerMeta}
        <button
          class="btn btn-sm btn-outline-secondary d-block w-100 mt-1"
          type="button"
          data-bs-toggle="collapse"
          data-bs-target="#modelMeta">
          Model metadata <MdIcon icon="expand_more" />
        </button>

        <div class="collapse" id="modelMeta">
          <ul>
            {#each Object.entries($printerMeta) as [key, value] (key)}
              <li>{key}: <strong>{value ?? "-"}</strong></li>
            {/each}
          </ul>
        </div>
      {/if}

      {#if $rfidInfo}
        <button
          class="btn btn-sm btn-outline-secondary d-block w-100 mt-1"
          type="button"
          data-bs-toggle="collapse"
          data-bs-target="#rfidInfo">
          RFID info <MdIcon icon="expand_more" />
        </button>

        <div class="collapse" id="rfidInfo">
          <button class="btn btn-outline-secondary btn-sm mt-1" onclick={refreshRfidInfo}>Update</button>

          <ul>
            {#each Object.entries($rfidInfo) as [key, value] (key)}
              <li>{key}: <strong>{value ?? "-"}</strong></li>
            {/each}
          </ul>
        </div>
      {/if}

      {#if $ribbonRfidInfo}
        <button
          class="btn btn-sm btn-outline-secondary d-block w-100 mt-1"
          type="button"
          data-bs-toggle="collapse"
          data-bs-target="#ribbonRfidInfo">
          Ribbon RFID info <MdIcon icon="expand_more" />
        </button>

        <div class="collapse" id="ribbonRfidInfo">
          <button class="btn btn-outline-secondary btn-sm mt-1" onclick={refreshRfidInfo}>Update</button>

          <ul>
            {#each Object.entries($ribbonRfidInfo) as [key, value] (key)}
              <li>{key}: <strong>{value ?? "-"}</strong></li>
            {/each}
          </ul>
        </div>
      {/if}

      {#if $heartbeatData}
        <button
          class="btn btn-sm btn-outline-secondary d-block w-100 mt-1"
          type="button"
          data-bs-toggle="collapse"
          data-bs-target="#heartbeatData">
          Heartbeat data <MdIcon icon="expand_more" />
        </button>

        <div class="collapse" id="heartbeatData">
          <ul>
            {#each Object.entries($heartbeatData) as [key, value] (key)}
              <li>{key}: <strong>{value ?? "-"}</strong></li>
            {/each}
          </ul>
        </div>
      {/if}

      <FirmwareUpdater />

      <button
        class="btn btn-sm btn-outline-secondary d-block w-100 mt-1"
        type="button"
        data-bs-toggle="collapse"
        data-bs-target="#tests">
        Tests <MdIcon icon="expand_more" />
      </button>

      <div class="collapse" id="tests">
        <div class="d-flex flex-wrap gap-1 mt-1">
          <button class="btn btn-sm btn-primary" onclick={startHeartbeat}>Heartbeat on</button>
          <button class="btn btn-sm btn-primary" onclick={stopHeartbeat}>Heartbeat off</button>
          <button class="btn btn-sm btn-primary" onclick={soundOn}>Sound on</button>
          <button class="btn btn-sm btn-primary" onclick={soundOff}>Sound off</button>
          <button class="btn btn-sm btn-primary" onclick={fetchInfo}>Fetch info again</button>
          <button class="btn btn-sm btn-primary" onclick={reset}>Reset</button>
        </div>
      </div>
    </div>
    <span class="input-group-text">
      {#if connectionType === "serial"}
        <MdIcon icon="usb" />
      {:else}
        <MdIcon icon="bluetooth" />
      {/if}
    </span>
    <span class="input-group-text {$heartbeatFails > 0 ? 'text-warning' : ''}">
      {$printerMeta?.model ?? $connectedPrinterName}
    </span>
    {#if $heartbeatData?.chargeLevel}
      <span class="input-group-text">
        <MdIcon icon={batteryIcon($heartbeatData.chargeLevel)} class="r-90"></MdIcon>
      </span>
    {/if}
  {:else}
    {#if featureSupport.webBluetooth}
      <button
        disabled={$connectionState === "connecting"}
        class="btn text-nowrap {connectionType === 'bluetooth' ? 'btn-light' : 'btn-outline-secondary'}"
        onclick={() => switchConnectionType("bluetooth")}>
        <MdIcon icon="bluetooth" />
        {$tr("connector.bluetooth")}
      </button>
    {/if}
    {#if featureSupport.webSerial}
      <button
        disabled={$connectionState === "connecting"}
        class="btn text-nowrap {connectionType === 'serial' ? 'btn-light' : 'btn-outline-secondary'}"
        onclick={() => switchConnectionType((connectionType = "serial"))}>
        <MdIcon icon="usb" />
        {$tr("connector.serial")}
      </button>
    {/if}
    {#if featureSupport.capacitorBle}
      <button
        disabled={$connectionState === "connecting"}
        class="btn text-nowrap {connectionType === 'capacitor-ble' ? 'btn-light' : 'btn-outline-secondary'}"
        onclick={() => switchConnectionType((connectionType = "capacitor-ble"))}>
        <MdIcon icon="usb" />
        Capacitor BLE
      </button>
    {/if}
  {/if}

  {#if $connectionState !== "connected"}
    <button
      class="btn btn-primary"
      disabled={$connectionState === "connecting" ||
        (!featureSupport.capacitorBle && !featureSupport.webBluetooth && !featureSupport.webSerial)}
      onclick={() => onConnectClicked(false)}>
      <MdIcon icon="power" />
    </button>
  {/if}

  {#if $connectionState === "connected"}
    <button class="btn btn-danger" onclick={onDisconnectClicked}>
      <MdIcon icon="power_off" />
    </button>
  {/if}
</div>

<style>
  .dropdown-menu {
    width: 100vw;
    max-width: 300px;
  }
</style>
</file>