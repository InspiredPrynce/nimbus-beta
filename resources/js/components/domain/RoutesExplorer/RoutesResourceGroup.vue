<script setup lang="ts">
/**
 * @component RoutesResourceGroup
 * @description A collapsible sidebar folder for a route resource. Renders
 *              recursively: a node like "creator/auth/password" becomes a
 *              folder tree (creator → auth → password) with routes on the
 *              folder they belong to.
 */
import { AppCollapsible } from '@/components/base/collapsible';
import AppIcon from '@/components/base/icons/AppIcon.vue';
import ExplorerRow from '@/components/domain/RoutesExplorer/ExplorerRow.vue';
import RoutesListItem from '@/components/domain/RoutesExplorer/RoutesList/RoutesListItem.vue';
import { useShowingSearchResults } from '@/components/domain/RoutesExplorer/showingSearchResults';
import { useExplorerVisibility } from '@/composables/ui/useExplorerVisibility';
import type { RouteDefinition } from '@/interfaces/routes/routes';
import { useRequestStore } from '@/stores';
import type { ResourceTreeNode } from '@/utils/routes/resourceTree';
import { singletonPersistenceKey } from '@/utils/stores/uniquePersistenceKey';
import { useStorage } from '@vueuse/core';
import { computed } from 'vue';

/*
 * Types & Interfaces.
 */

export interface RoutesResourceGroupProps {
    /** The folder node: name, full resource path, routes and children. */
    node: ResourceTreeNode;
}

/*
 * Component Setup.
 */

const props = defineProps<RoutesResourceGroupProps>();

/*
 * Stores & Dependencies.
 */

const requestStore = useRequestStore();
const { isExplorerDrawerOpen } = useExplorerVisibility();

/*
 * State.
 */

const isOpen = useStorage(
    singletonPersistenceKey(`routes-explorer-resource-${props.node.fullPath}-expanded`),
    false,
);

const showingSearchResults = useShowingSearchResults();

/*
 * Computed & Methods.
 */

const pendingRequestData = computed(() => requestStore.pendingRequestData);

const isRouteActive = (route: RouteDefinition) => {
    if (!route) {
        return false;
    }
    if (!pendingRequestData.value) {
        return false;
    }

    return (
        route.endpoint === pendingRequestData.value.endpoint &&
        route.method === pendingRequestData.value.method
    );
};

const setPendingRequest = (route: RouteDefinition) => {
    if (!route) {
        return;
    }

    // Get all routes with the same endpoint (different HTTP methods).
    const availableRoutesForEndpoint = props.node.routes.filter(
        (routeInGroup: RouteDefinition) => routeInGroup.endpoint === route.endpoint,
    );

    requestStore.initializeRequest(route, availableRoutesForEndpoint);
    isExplorerDrawerOpen.value = false;
};
</script>

<template>
    <li>
        <AppCollapsible
            class="group/collapsible"
            :default-open="isOpen"
            :open="showingSearchResults ? true : isOpen"
            @update:open="isOpen = $event"
        >
            <ExplorerRow>
                <AppIcon
                    name="i-hugeicons-arrow-right-01"
                    class="-mx-tight shrink-0 transition-transform group-data-[state=open]/collapsible:rotate-90"
                />
                <AppIcon name="i-hugeicons-folder-01" class="shrink-0" />
                <span class="max-w-45 truncate sm:max-w-55 md:max-w-65 lg:max-w-75">
                    {{ node.name }}
                </span>
            </ExplorerRow>
            <template #content>
                <ul
                    class="border-default ms-loose gap-tight py-tight ps-tight flex min-w-0 flex-col border-s"
                >
                    <!-- Nested child folders (recursive). -->
                    <RoutesResourceGroup
                        v-for="child in node.children"
                        :key="child.fullPath"
                        :node="child"
                    />
                    <!-- Routes directly in this folder. -->
                    <RoutesListItem
                        v-for="route in node.routes"
                        :key="route.endpoint + '-' + route.method"
                        :route="route"
                        :resource="node.fullPath"
                        :on-click="() => setPendingRequest(route)"
                        :is-active="isRouteActive(route)"
                    />
                </ul>
            </template>
        </AppCollapsible>
    </li>
</template>
