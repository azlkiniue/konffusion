<script lang="ts">
	import YAML from 'yaml';
	import * as Card from '../ui/card';
	import * as Dialog from '../ui/dialog';
	import * as Select from '../ui/select';
	import * as Tabs from '../ui/tabs';
	import { parse } from '../common/handle-config-yaml';
	import { buttonVariants, Button } from '../ui/button';
	import { Input } from '../ui/input';
	import { Label } from '../ui/label';
	import { ScrollArea } from '../ui/scroll-area';
	import {
		checkLength,
		validateKubeConfig,
		type KubernetesConfig,
		type Cluster,
		type User,
		type Context
	} from '$lib/models/kubeconfig';
	import { base, added, merged } from '$lib/stores';
	import { Pencil2, Cross2, Check } from 'svelte-radix';

	export let config: string;
	export let typeConfig: string;
	let parsedConfig: KubernetesConfig;
	let isValidKubeconfig: boolean;
	let previousNames: Record<string, string[]>;
	let editData: {
		cluster: Array<Cluster | undefined>;
		user: Array<User | undefined>;
		context: Array<Context | undefined>;
	} = { cluster: [], user: [], context: [] };

	function parseConfig() {
		try {
			parsedConfig = parse(config);
			isValidKubeconfig = validateKubeConfig(YAML.parse(config));
			previousNames = {
				cluster: parsedConfig.clusters.map((cluster) => cluster.name),
				user: parsedConfig.users.map((user) => user.name)
			};
		} catch (e) {
			isValidKubeconfig = false;
		}
	}

	function handleNameChangeContext(
		index: number,
		newName: string,
		resourceType: 'cluster' | 'user'
	) {
		const oldName = previousNames[resourceType][index];
		parsedConfig[(resourceType + 's') as 'clusters' | 'users'][index].name = newName;
		parsedConfig.contexts.forEach((context) => {
			if (context.context[resourceType] === oldName) {
				context.context[resourceType] = newName;
			}
		});
		previousNames[resourceType][index] = newName;

		editData.context.forEach((context) => {
			if (context?.context[resourceType] === oldName) {
				context.context[resourceType] = newName;
			}
		});
	}

	function handleStartEdit(editType: 'cluster' | 'user' | 'context', index: number) {
		if (!isValidKubeconfig) {
			return;
		}
		if (editType === 'cluster') {
			editData.cluster[index] = structuredClone(parsedConfig.clusters[index]);
		} else if (editType === 'user') {
			editData.user[index] = structuredClone(parsedConfig.users[index]);
		} else if (editType === 'context') {
			editData.context[index] = structuredClone(parsedConfig.contexts[index]);
		}
	}

	function handleSaveEdit(editType: 'cluster' | 'user' | 'context', index: number) {
		if (!isValidKubeconfig) {
			return;
		}
		if (editType === 'cluster' && editData.cluster[index]) {
			parsedConfig.clusters[index] = structuredClone(editData.cluster[index]);
			editData.cluster[index] = undefined;
			handleNameChangeContext(index, parsedConfig.clusters[index].name, 'cluster');
		} else if (editType === 'user' && editData.user[index]) {
			parsedConfig.users[index] = structuredClone(editData.user[index]);
			editData.user[index] = undefined;
			handleNameChangeContext(index, parsedConfig.users[index].name, 'user');
		} else if (editType === 'context' && editData.context[index]) {
			parsedConfig.contexts[index] = structuredClone(editData.context[index]);
			editData.context[index] = undefined;
		}
	}

	function save() {
		if (typeConfig === 'base') {
			$base = YAML.stringify(parsedConfig);
		}
		if (typeConfig === 'merged') {
			$merged = YAML.stringify(parsedConfig);
		}
		if (typeConfig.startsWith('added')) {
			const index = parseInt(typeConfig.split('-')[1]);
			$added[index] = YAML.stringify(parsedConfig);
		}
	}
</script>

<Dialog.Root>
	<Dialog.Trigger class={buttonVariants({ variant: 'secondary' })} on:click={() => parseConfig()}>
		<Pencil2 class="mr-2 h-4 w-4" />
		Easy Edit
	</Dialog.Trigger>
	<Dialog.Content class="max-h-[80%] max-w-2xl overflow-y-auto">
		<Dialog.Header>
			<Dialog.Title>Easy Edit Kubeconfig</Dialog.Title>
			<Dialog.Description>
				Easily edit your kubeconfig here, just adjust the values using the form below and click save
				changes.
			</Dialog.Description>
		</Dialog.Header>
		<div class="grid gap-4 py-4">
			{#if isValidKubeconfig}
				<Tabs.Root value="cluster" class="w-full">
					<Tabs.List class="mb-4 grid grid-cols-3">
						<Tabs.Trigger value="cluster">
							Clusters ({checkLength(parsedConfig.clusters)})
						</Tabs.Trigger>
						<Tabs.Trigger value="user">Users ({checkLength(parsedConfig.users)})</Tabs.Trigger>
						<Tabs.Trigger value="context">
							Contexts ({checkLength(parsedConfig.contexts)})
						</Tabs.Trigger>
					</Tabs.List>
					<Tabs.Content value="cluster">
						<div class="space-y-4">
							<div class="flex items-center justify-between">
								<h3 class="text-lg font-medium">Clusters</h3>
							</div>
						</div>

						<ScrollArea class="mt-4 h-[300px] rounded-md border p-4"> <!-- clusters -->
							<div class="space-y-4">
								{#each parsedConfig.clusters as cluster, key}
									<Card.Root class="p-4">
										{#if editData.cluster[key]}
											<div class="flex w-full flex-col gap-1.5">
												<Label for="cls-{key}" class="text-left text-base">Name</Label>
												<Input
													id="cls-{key}"
													bind:value={editData.cluster[key].name}
													class="col-span-3"
												/>
												<Label for="cls-addr-{key}" class="text-left text-base">Address</Label>
												<Input
													id="cls-addr-{key}"
													bind:value={editData.cluster[key].cluster.server}
													class="col-span-3"
												/>
											</div>
											<div class="mt-2 flex justify-end gap-2">
												<Button
													variant="ghost"
													size="sm"
													on:click={() => (editData.cluster[key] = undefined)}
												>
													<Cross2 class="mr-1 h-4 w-4" />
													Cancel
												</Button>
												<Button
													variant="default"
													size="sm"
													on:click={() => handleSaveEdit('cluster', key)}
												>
													<Check class="mr-1 h-4 w-4" />
													Done
												</Button>
											</div>
										{:else}
											<div class="flex items-start justify-between">
												<div>
													<h4 class="font-medium">{cluster.name}</h4>
													<p class="text-sm text-muted-foreground">{cluster.cluster.server}</p>
												</div>
												<div class="flex gap-1">
													<Button on:click={() => handleStartEdit('cluster', key)}>Edit</Button>
													<!-- <Button>Delete</Button> TODO: Implement delete -->
												</div>
											</div>
										{/if}
									</Card.Root>
								{/each}
							</div>
						</ScrollArea>
					</Tabs.Content>
					<Tabs.Content value="user">
						<div class="space-y-4">
							<div class="flex items-center justify-between">
								<h3 class="text-lg font-medium">Users</h3>
							</div>
						</div>
						<ScrollArea class="mt-4 h-[300px] rounded-md border p-4"> <!-- users -->
							<div class="space-y-4">
								{#each parsedConfig.users as user, key}
									<Card.Root class="p-4">
										{#if editData.user[key]}
											<div class="flex w-full flex-col gap-1.5">
												<Label for="usr-{key}" class="text-left text-base">Name</Label>
												<Input
													id="usr-{key}"
													bind:value={editData.user[key].name}
													class="col-span-3"
												/>
											</div>
											<div class="mt-2 flex justify-end gap-2">
												<Button
													variant="ghost"
													size="sm"
													on:click={() => (editData.user[key] = undefined)}
												>
													<Cross2 class="mr-1 h-4 w-4" />
													Cancel
												</Button>
												<Button
													variant="default"
													size="sm"
													on:click={() => handleSaveEdit('user', key)}
												>
													<Check class="mr-1 h-4 w-4" />
													Done
												</Button>
											</div>
										{:else}
											<div class="flex items-center justify-between">
												<div>
													<h4 class="font-medium">{user.name}</h4>
												</div>
												<div class="flex gap-1">
													<Button size="sm" on:click={() => handleStartEdit('user', key)}>
														Edit
													</Button>
													<!-- <Button size="sm">Delete</Button> TODO: Implement delete -->
												</div>
											</div>
										{/if}
									</Card.Root>
								{/each}
							</div>
						</ScrollArea>
					</Tabs.Content>
					<Tabs.Content value="context">
						<div class="space-y-4">
							<div class="flex items-center justify-between">
								<h3 class="text-lg font-medium">Contexts</h3>
							</div>
						</div>
						<ScrollArea class="mt-4 h-[300px] rounded-md border p-4"> <!-- contexts -->
							<div class="space-y-4">
								{#each parsedConfig.contexts as context, key}
									<Card.Root class="p-4">
										{#if editData.context[key]}
											<div class="flex w-full flex-col gap-1.5">
												<Label for="ctx-{key}" class="text-left text-base">Name</Label>
												<Input
													id="ctx-{key}"
													bind:value={editData.context[key].name}
													class="col-span-3"
												/>
												<Label for="ctx-cls-{key}" class="text-left text-base">Cluster</Label>
												<Select.Root
													portal={null}
													selected={{
														value: editData.context[key].context.cluster,
														label: editData.context[key].context.cluster
													}}
													onSelectedChange={(e) => {
														e &&
															editData.context[key] &&
															(editData.context[key].context.cluster = e.value);
													}}
												>
													<Select.Trigger class="col-span-3 [&>span]:truncate">
														<Select.Value placeholder="Select a cluster" />
													</Select.Trigger>
													<Select.Content>
														<Select.Group>
															{#each parsedConfig.clusters as cluster}
																<Select.Item bind:value={cluster.name} bind:label={cluster.name}>
																	{cluster.name}
																</Select.Item>
															{/each}
														</Select.Group>
													</Select.Content>
													<Select.Input id="ctx-cls-{key}" />
												</Select.Root>
												<Label for="ctx-usr-{key}" class="text-left text-base">User</Label>
												<Select.Root
													portal={null}
													selected={{
														value: editData.context[key].context.user,
														label: editData.context[key].context.user
													}}
													onSelectedChange={(e) => {
														e &&
															editData.context[key] &&
															(editData.context[key].context.user = e.value);
													}}
												>
													<Select.Trigger class="col-span-3 [&>span]:truncate">
														<Select.Value placeholder="Select a user" />
													</Select.Trigger>
													<Select.Content>
														<Select.Group>
															{#each parsedConfig.users as user}
																<Select.Item bind:value={user.name} bind:label={user.name}>
																	{user.name}
																</Select.Item>
															{/each}
														</Select.Group>
													</Select.Content>
													<Select.Input id="ctx-usr-{key}" />
												</Select.Root>
											</div>
											<div class="mt-2 flex justify-end gap-2">
												<Button
													variant="ghost"
													size="sm"
													on:click={() => (editData.context[key] = undefined)}
												>
													<Cross2 class="mr-1 h-4 w-4" />
													Cancel
												</Button>
												<Button
													variant="default"
													size="sm"
													on:click={() => handleSaveEdit('context', key)}
												>
													<Check class="mr-1 h-4 w-4" />
													Done
												</Button>
											</div>
										{:else}
											<div class="flex items-start justify-between">
												<div>
													<h4 class="font-medium">{context.name}</h4>
													<p class="text-sm text-muted-foreground">
														Cluster: {context.context.cluster || 'Unknown'}
														| User: {context.context.user || 'Unknown'}
													</p>
												</div>
												<div class="flex gap-1">
													<Button on:click={() => handleStartEdit('context', key)}>Edit</Button>
													<!-- <Button>Delete</Button> TODO: Implement delete -->
												</div>
											</div>
										{/if}
									</Card.Root>
								{/each}
							</div>
						</ScrollArea>
					</Tabs.Content>
				</Tabs.Root>
			{:else}
				<p class="text-xl text-destructive">Kubeconfig is not valid or empty</p>
			{/if}
		</div>
		<Dialog.Footer>
			<Dialog.Close class={buttonVariants({ variant: 'outline' })}>Cancel</Dialog.Close>
			<Dialog.Close
				disabled={!isValidKubeconfig}
				class={buttonVariants({ variant: 'default' })}
				on:click={() => save()}
			>
				Save changes
			</Dialog.Close>
		</Dialog.Footer>
	</Dialog.Content>
</Dialog.Root>
