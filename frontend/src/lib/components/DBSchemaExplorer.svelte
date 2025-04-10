<script lang="ts">
	import { dbSchemas, workspaceStore, type DBSchema } from '$lib/stores'
	import Button from './common/button/Button.svelte'
	import Drawer from './common/drawer/Drawer.svelte'
	import DrawerContent from './common/drawer/DrawerContent.svelte'
	import ObjectViewer from './propertyPicker/ObjectViewer.svelte'
	import { sendUserToast } from '$lib/utils'
	import ToggleButton from './common/toggleButton-v2/ToggleButton.svelte'
	import ToggleButtonGroup from './common/toggleButton-v2/ToggleButtonGroup.svelte'
	import { Loader2, RefreshCcw } from 'lucide-svelte'
	import {
		formatGraphqlSchema,
		formatSchema,
		getDbSchemas,
		scripts
	} from './apps/components/display/dbtable/utils'
	import Alert from './common/alert/Alert.svelte'

	export let resourceType: string | undefined
	export let resourcePath: string | undefined = undefined
	let dbSchema: DBSchema | undefined = undefined
	let loading = false

	let drawer: Drawer | undefined

	async function getSchema() {
		if (!resourceType || !resourcePath) return
		if ($dbSchemas[resourcePath]) return

		loading = true

		try {
			await getDbSchemas(
				resourceType,
				resourcePath,
				$workspaceStore,
				$dbSchemas,
				(message: string) => {
					if (drawer?.isOpen()) {
						sendUserToast(message, true)
					}
				}
			)
			$dbSchemas = $dbSchemas
		} catch (e) {
			console.error(e)
		}
		loading = false
	}

	$: resourcePath && Object.keys(scripts).includes(resourceType || '') && getSchema()

	$: dbSchema = resourcePath && resourcePath in $dbSchemas ? $dbSchemas[resourcePath] : undefined

	$: shouldDisplayError = resourcePath && resourcePath in $dbSchemas && !$dbSchemas[resourcePath]

	function handleSelected({ detail }: CustomEvent<string>) {
		if (dbSchema && dbSchema.lang !== 'graphql') {
			dbSchema.publicOnly = detail === 'dbo'
		}
	}
</script>

{#if loading}
	<Loader2 size={14} class="animate-spin " />
{/if}

{#if dbSchema}
	<Button
		size="xs"
		variant="border"
		color="blue"
		spacingSize="xs2"
		btnClasses="mt-1"
		on:click={drawer?.openDrawer ?? (() => {})}
	>
		Explore schema
	</Button>
	<Drawer bind:this={drawer}>
		<DrawerContent title="Schema Explorer" on:close={drawer.closeDrawer}>
			<svelte:fragment slot="actions">
				<Button
					on:click={getSchema}
					startIcon={{
						icon: RefreshCcw
					}}
					{loading}
					size="xs"
					color="light"
				>
					Refresh
				</Button>
			</svelte:fragment>
			{#if dbSchema.lang !== 'graphql' && (dbSchema.schema?.public || dbSchema.schema?.PUBLIC || dbSchema.schema?.dbo)}
				<ToggleButtonGroup
					class="mb-4"
					selected={dbSchema.publicOnly ? 'dbo' : 'all'}
					on:selected={handleSelected}
					let:item
				>
					<ToggleButton value="dbo" label={dbSchema.schema.dbo ? 'Dbo' : 'Public'} {item} />
					<ToggleButton value="all" label="All" {item} />
				</ToggleButtonGroup>
			{/if}
			{#if dbSchema.lang === 'graphql'}
				{#await import('$lib/components/GraphqlSchemaViewer.svelte')}
					<Loader2 class="animate-spin" />
				{:then Module}
					<Module.default code={formatGraphqlSchema(dbSchema.schema)} class="h-full" />
				{/await}
			{:else}
				<ObjectViewer json={formatSchema(dbSchema)} pureViewer collapseLevel={1} />
			{/if}
		</DrawerContent>
	</Drawer>
{:else if shouldDisplayError}
	<Alert type="error" size="xs" title="Schema not available" class="mt-2">
		Schema could not be loaded. Please check the permissions of the resource.
	</Alert>
{/if}
