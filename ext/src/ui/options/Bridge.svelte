<script lang="ts">
    import semver from "semver";
    import { onMount } from "svelte";

    import LoadingIndicator from "../LoadingIndicator.svelte";

    import bridge, { BridgeInfo, BridgeTimedOutError } from "../../lib/bridge";
    import logger from "../../lib/logger";

    import { Options } from "../../lib/options";

    const _ = browser.i18n.getMessage;

    export let opts: Options;

    let bridgeInfo: Nullable<BridgeInfo> = null;
    let isLoadingInfo = true;
    let isLoadingInfoTimedOut = false;

    // Status
    let infoClass = "bridge__info";
    let statusIcon: string;
    let statusTitle: string;
    let statusText: Nullable<string> = null;

    onMount(async () => {
        try {
            bridgeInfo = await bridge.getInfo();
        } catch (err) {
            logger.error("Failed to fetch bridge/platform info.");
            if (err instanceof BridgeTimedOutError) {
                isLoadingInfoTimedOut = true;
            }
        }

        isLoadingInfo = false;

        infoClass += ` ${
            !bridgeInfo
                ? isLoadingInfoTimedOut
                    ? "bridge__info--timed-out"
                    : "bridge__info--not-found"
                : "bridge__info--found"
        }`;

        if (!bridgeInfo) {
            statusIcon = "assets/icons8-cancel-120.png";
            statusTitle = _("optionsBridgeNotFoundStatusTitle");
            statusText = _("optionsBridgeNotFoundStatusText");
        } else if (isLoadingInfoTimedOut) {
            statusIcon = "assets/icons8-warn-120.png";
            statusTitle = _("optionsBridgeIssueStatusTitle");
        } else {
            if (bridgeInfo.isVersionCompatible) {
                statusIcon = "assets/icons8-ok-120.png";
                statusTitle = _("optionsBridgeFoundStatusTitle");
            } else {
                statusIcon = "assets/icons8-warn-120.png";
                statusTitle = _("optionsBridgeIssueStatusTitle");
            }
        }
    });

    // Updates
    let updateData: Nullable<GitHubRelease> = null;
    let updateStatus: Nullable<string> = null;
    let updateStatusTimeout: number;

    let isCheckingUpdate = false;
    let isUpdateAvailable = false;

    interface GitHubRelease {
        url: string;
        tag_name: string;
        html_url: string;
        assets: Array<{
            content_type: string;
            html_url: string;
        }>;
    }

    async function checkUpdate() {
        isCheckingUpdate = true;

        let releases: GitHubRelease[];
        try {
            releases = await fetch(
                "https://api.github.com/repos/hensm/fx_cast/releases"
            ).then(res => res.json());
        } catch (err) {
            isCheckingUpdate = false;
            updateStatus = _("optionsBridgeUpdateStatusError");
            return;
        }

        // Ensure valid response
        if (!Array.isArray(releases)) {
            throw logger.error("Check update response is not array.", releases);
        }

        // First non-extension-only release
        const latestBridgeRelease = releases.find(release =>
            release.assets.find(
                asset => asset.content_type !== "application/x-xpinstall"
            )
        );

        if (!latestBridgeRelease) {
            throw logger.error(
                "Check update response does not contain release info."
            );
        }

        /**
         * Update available if no bridge found or bridge version lower
         * than fetched release version.
         */
        isUpdateAvailable =
            !bridgeInfo ||
            semver.lt(bridgeInfo.version, latestBridgeRelease.tag_name);

        if (isUpdateAvailable) {
            updateData = latestBridgeRelease;
        } else {
            updateStatus = _("optionsBridgeUpdateStatusNoUpdates");
        }

        isCheckingUpdate = false;

        if (updateStatusTimeout) {
            window.clearTimeout(updateStatusTimeout);
        }
        updateStatusTimeout = window.setTimeout(() => {
            updateStatus = null;
        }, 1500);
    }

    function getUpdate() {
        // Open downloads page
        if (updateData?.html_url) {
            browser.tabs.create({ url: updateData.html_url });
        }
    }

    const [backupMessageStart, backupMessageEnd] = _(
        "optionsBridgeBackupEnabled",
        "\0"
    ).split("\0");
</script>

<div class="bridge">
    {#if isLoadingInfo}
        <div class="bridge__loading">
            {_("optionsBridgeLoading")}
            <progress />
        </div>
    {:else}
        <div class={infoClass}>
            <div class="bridge__status">
                <img
                    class="bridge__status-icon"
                    width="60"
                    height="60"
                    src={statusIcon}
                    alt="icon, bridge status"
                />
                <h2 class="bridge__status-title">{statusTitle}</h2>
                {#if statusText}
                    <p class="bridge__status-text">{statusText}</p>
                {/if}
            </div>

            {#if bridgeInfo}
                <table class="bridge__stats">
                    <tr>
                        <th>{_("optionsBridgeStatsName")}</th>
                        <td>{bridgeInfo.name}</td>
                    </tr>
                    <tr>
                        <th>{_("optionsBridgeStatsVersion")}</th>
                        <td>{bridgeInfo.version}</td>
                    </tr>
                    <tr>
                        <th>{_("optionsBridgeStatsExpectedVersion")}</th>
                        <td>{bridgeInfo.expectedVersion}</td>
                    </tr>
                    <tr>
                        <th>{_("optionsBridgeStatsCompatibility")}</th>
                        <td>
                            {bridgeInfo.isVersionCompatible
                                ? bridgeInfo.isVersionExact
                                    ? _("optionsBridgeCompatible")
                                    : _("optionsBridgeLikelyCompatible")
                                : _("optionsBridgeIncompatible")}
                        </td>
                    </tr>
                    <tr>
                        <th>{_("optionsBridgeStatsRecommendedAction")}</th>
                        <td>
                            {bridgeInfo.isVersionCompatible
                                ? _("optionsBridgeNoAction")
                                : bridgeInfo.isVersionOlder
                                ? _("optionsBridgeOlderAction")
                                : bridgeInfo.isVersionNewer
                                ? _("optionsBridgeNewerAction")
                                : _("optionsBridgeNoAction")}
                        </td>
                    </tr>
                </table>
            {/if}
        </div>

        <div class="bridge__options">
            <div class="option option--inline">
                <div class="option__control">
                    <input
                        name="bridgeBackupEnabled"
                        id="bridgeBackupEnabled"
                        type="checkbox"
                        bind:checked={opts.bridgeBackupEnabled}
                    />
                </div>
                <label class="option__label" for="bridgeBackupEnabled">
                    {backupMessageStart}
                    <input
                        class="bridge__backup-host"
                        name="bridgeBackupHost"
                        type="text"
                        required
                        bind:value={opts.bridgeBackupHost}
                    />
                    :
                    <input
                        class="bridge__backup-port"
                        name="bridgeBackupPort"
                        type="number"
                        required
                        min="1025"
                        max="65535"
                        bind:value={opts.bridgeBackupPort}
                    />
                    {backupMessageEnd}
                </label>
                <div class="option__description">
                    {_("optionsBridgeBackupEnabledDescription")}
                </div>
            </div>
        </div>

        <div class="bridge__update-info">
            {#if isUpdateAvailable}
                <div class="bridge__update">
                    <p class="bridge__update-label">
                        {_("optionsBridgeUpdateAvailable")}
                    </p>
                    <div class="bridge__update-options">
                        <button
                            class="bridge__update-start"
                            type="button"
                            on:click={getUpdate}
                        >
                            {_("optionsBridgeUpdate")}
                        </button>
                    </div>
                </div>
            {:else}
                <button
                    class="bridge__update-check"
                    type="button"
                    disabled={isCheckingUpdate}
                    on:click={checkUpdate}
                >
                    {#if isCheckingUpdate}
                        {_("optionsBridgeUpdateChecking", "")}<LoadingIndicator
                        />
                    {:else}
                        {_("optionsBridgeUpdateCheck")}
                    {/if}
                </button>
            {/if}

            {#if updateStatus && !isUpdateAvailable}
                <div class="bridge--update-status">
                    {updateStatus}
                </div>
            {/if}
        </div>
    {/if}
</div>
