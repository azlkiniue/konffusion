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
	import { SquarePen, X, Save, Trash2 } from '@lucide/svelte';

	interface Props {
		config: string;
		typeConfig: string;
	}

	let { config, typeConfig }: Props = $props();
	let parsedConfig: KubernetesConfig | undefined = $state();
	let isValidKubeconfig: boolean = $state(false);
	let previousNames: Record<string, string[]>;
	let editData: {
		cluster: Array<Cluster | undefined>;
		user: Array<User | undefined>;
		context: Array<Context | undefined>;
	} = $state({ cluster: [], user: [], context: [] });
	let deleteData: {
		cluster: Array<Cluster | undefined>;
		user: Array<User | undefined>;
		context: Array<Context | undefined>;
	} = $state({ cluster: [], user: [], context: [] });

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
		if (!parsedConfig) return;

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
		if (!isValidKubeconfig || !parsedConfig) {
			return;
		}
		let parsedConfigSnapshot: KubernetesConfig = structuredClone($state.snapshot(parsedConfig));
		if (editType === 'cluster') {
			editData.cluster[index] = parsedConfigSnapshot.clusters[index];
		} else if (editType === 'user') {
			editData.user[index] = parsedConfigSnapshot.users[index];
		} else if (editType === 'context') {
			editData.context[index] = parsedConfigSnapshot.contexts[index];
		}
	}

	function handleStartDelete(deleteType: 'cluster' | 'user' | 'context', index: number) {
		if (!isValidKubeconfig || !parsedConfig) {
			return;
		}
		if (deleteType === 'cluster') {
			deleteData.cluster[index] = parsedConfig.clusters[index];
		} else if (deleteType === 'user') {
			deleteData.user[index] = parsedConfig.users[index];
		} else if (deleteType === 'context') {
			deleteData.context[index] = parsedConfig.contexts[index];
		}
	}

	function handleSaveEdit(editType: 'cluster' | 'user' | 'context', index: number) {
		if (!isValidKubeconfig || !parsedConfig) {
			return;
		}
		let editDataSnapshot = structuredClone($state.snapshot(editData));
		if (editType === 'cluster' && editData.cluster[index]) {
			parsedConfig.clusters[index] = editDataSnapshot.cluster[index] as Cluster;
			editData.cluster[index] = undefined;
			handleNameChangeContext(index, parsedConfig.clusters[index].name, 'cluster');
		} else if (editType === 'user' && editData.user[index]) {
			parsedConfig.users[index] = editDataSnapshot.user[index] as User;
			editData.user[index] = undefined;
			handleNameChangeContext(index, parsedConfig.users[index].name, 'user');
		} else if (editType === 'context' && editData.context[index]) {
			parsedConfig.contexts[index] = editDataSnapshot.context[index] as Context;
			editData.context[index] = undefined;
		}
	}

	function handleSaveDelete(
		editType: 'cluster' | 'user' | 'context',
		index: number,
		isDeleteAll: boolean = false
	) {
		if (!isValidKubeconfig || !parsedConfig) {
			return;
		}
		if (editType === 'cluster' && deleteData.cluster[index]) {
			parsedConfig.clusters.splice(index, 1);
			if (isDeleteAll) {
				// Remove all contexts that reference this cluster
				parsedConfig.contexts = parsedConfig.contexts.filter(
					(context) => context.context.cluster !== (deleteData.cluster[index] as Cluster).name
				);
			}
			deleteData.cluster[index] = undefined;
		} else if (editType === 'user' && deleteData.user[index]) {
			parsedConfig.users.splice(index, 1);
			if (isDeleteAll) {
				// Remove all contexts that reference this user
				parsedConfig.contexts = parsedConfig.contexts.filter(
					(context) => context.context.user !== (deleteData.user[index] as User).name
				);
			}
			deleteData.user[index] = undefined;
		} else if (editType === 'context' && deleteData.context[index]) {
			parsedConfig.contexts.splice(index, 1);
			if (isDeleteAll) {
				// Remove related clusters and users
				const context = deleteData.context[index] as Context;
				parsedConfig.clusters = parsedConfig.clusters.filter(
					(cluster) => cluster.name !== context.context.cluster
				);
				parsedConfig.users = parsedConfig.users.filter(
					(user) => user.name !== context.context.user
				);
			}
			deleteData.context[index] = undefined;
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
	<Dialog.Trigger class={buttonVariants({ variant: 'secondary' })} onclick={() => parseConfig()}>
		<SquarePen />
		Easy Edit
	</Dialog.Trigger>
	<Dialog.Content class="max-h-8/10 max-w-2xl! overflow-y-auto">
		<Dialog.Header>
			<Dialog.Title>Easy Edit Kubeconfig</Dialog.Title>
			<Dialog.Description>
				Easily edit your kubeconfig here, just adjust the values using the form below and click save
				changes.
			</Dialog.Description>
		</Dialog.Header>
		<div class="grid gap-4 py-4">
			{#if isValidKubeconfig && parsedConfig}
				<Tabs.Root value="cluster">
					<Tabs.List class="mb-4 grid grid-cols-3 w-full">
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

						<ScrollArea class="mt-4 h-[300px] rounded-md border p-4" id="clusters-scrollarea">
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
													onclick={() => (editData.cluster[key] = undefined)}
												>
													<X />
													Cancel
												</Button>
												<Button
													variant="default"
													size="sm"
													onclick={() => handleSaveEdit('cluster', key)}
												>
													<Save />
													Save
												</Button>
											</div>
										{:else if deleteData.cluster[key]}
											<div class="flex w-full flex-col gap-1">
												<p class="text-red-500 font-medium">
													Are you sure you want to delete this cluster?
												</p>
												<p>
													Cluster Name:
													<span class="font-semibold">
														{deleteData.cluster[key].name}
													</span>
												</p>
												<div class="mt-2 flex justify-end gap-2">
													<Button
														variant="ghost"
														size="sm"
														onclick={() => (deleteData.cluster[key] = undefined)}
													>
														<X />
														Cancel
													</Button>
													<Button
														variant="destructive"
														size="sm"
														onclick={() => handleSaveDelete('cluster', key, true)}
													>
														<Trash2 />
														Delete, with related contexts
													</Button>
													<Button
														variant="destructive"
														size="sm"
														onclick={() => handleSaveDelete('cluster', key)}
													>
														<Trash2 />
														Delete
													</Button>
												</div>
											</div>
										{:else}
											<div class="flex items-center justify-between">
												<div>
													<h4 class="font-medium">{cluster.name}</h4>
													<p class="text-sm text-muted-foreground">{cluster.cluster.server}</p>
												</div>
												<div class="flex gap-1">
													<Button onclick={() => handleStartEdit('cluster', key)}>Edit</Button>
													<Button
														variant="destructive"
														onclick={() => handleStartDelete('cluster', key)}>Delete</Button
													>
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
						<ScrollArea class="mt-4 h-[300px] rounded-md border p-4" id="users-scrollarea">
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
													onclick={() => (editData.user[key] = undefined)}
												>
													<X />
													Cancel
												</Button>
												<Button
													variant="default"
													size="sm"
													onclick={() => handleSaveEdit('user', key)}
												>
													<Save />
													Save
												</Button>
											</div>
										{:else if deleteData.user[key]}
											<div class="flex w-full flex-col gap-1">
												<p class="text-red-500 font-medium">
													Are you sure you want to delete this user?
												</p>
												<p>
													User Name: <span class="font-semibold">{deleteData.user[key].name}</span>
												</p>
												<div class="mt-2 flex justify-end gap-2">
													<Button
														variant="ghost"
														size="sm"
														onclick={() => (deleteData.user[key] = undefined)}
													>
														<X />
														Cancel
													</Button>
													<Button
														variant="destructive"
														size="sm"
														onclick={() => handleSaveDelete('user', key, true)}
													>
														<Trash2 />
														Delete, with related contexts
													</Button>
													<Button
														variant="destructive"
														size="sm"
														onclick={() => handleSaveDelete('user', key)}
													>
														<Trash2 />
														Delete
													</Button>
												</div>
											</div>
										{:else}
											<div class="flex items-center justify-between">
												<div>
													<h4 class="font-medium">{user.name}</h4>
												</div>
												<div class="flex gap-1">
													<Button size="sm" onclick={() => handleStartEdit('user', key)}>
														Edit
													</Button>
													<Button
														variant="destructive"
														size="sm"
														onclick={() => handleStartDelete('user', key)}
													>
														Delete
													</Button>
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
						<ScrollArea class="mt-4 h-[300px] rounded-md border p-4" id="contexts-scrollarea">
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
												<!-- placeholder to remove label warning -->
												<input id="ctx-cls-{key}" type="text" hidden />
												<Select.Root
													type="single"
													bind:value={editData.context[key].context.cluster}
													onValueChange={(value) => {
														value &&
															editData.context[key] &&
															(editData.context[key].context.cluster = value);
													}}
												>
													<Select.Trigger
														class="w-full col-span-3 [&>span]:truncate"
														placeholder="Select a cluster"
													>
														{editData.context[key].context.cluster}
													</Select.Trigger>
													<Select.Content>
														<Select.Group>
															{#each parsedConfig.clusters as cluster}
																<Select.Item value={cluster.name} label={cluster.name}>
																	{cluster.name}
																</Select.Item>
															{/each}
														</Select.Group>
													</Select.Content>
												</Select.Root>
												<Label for="ctx-usr-{key}" class="text-left text-base">User</Label>
												<!-- placeholder to remove label warning -->
												<input id="ctx-usr-{key}" type="text" hidden />
												<Select.Root
													type="single"
													bind:value={editData.context[key].context.user}
													onValueChange={(value) => {
														value &&
															editData.context[key] &&
															(editData.context[key].context.user = value);
													}}
												>
													<Select.Trigger
														class="w-full col-span-3 [&>span]:truncate"
														placeholder="Select a user"
													>
														{editData.context[key].context.user}
													</Select.Trigger>
													<Select.Content>
														<Select.Group>
															{#each parsedConfig.users as user}
																<Select.Item value={user.name} label={user.name}>
																	{user.name}
																</Select.Item>
															{/each}
														</Select.Group>
													</Select.Content>
												</Select.Root>
											</div>
											<div class="mt-2 flex justify-end gap-2">
												<Button
													variant="ghost"
													size="sm"
													onclick={() => (editData.context[key] = undefined)}
												>
													<X />
													Cancel
												</Button>
												<Button
													variant="default"
													size="sm"
													onclick={() => handleSaveEdit('context', key)}
												>
													<Save />
													Save
												</Button>
											</div>
										{:else if deleteData.context[key]}
											<div class="flex w-full flex-col gap-1">
												<p class="text-red-500 font-medium">
													Are you sure you want to delete this context?
												</p>
												<p>
													Context Name:
													<span class="font-semibold">
														{deleteData.context[key].name}
													</span>
												</p>
												<div class="mt-2 flex justify-end gap-2">
													<Button
														variant="ghost"
														size="sm"
														onclick={() => (deleteData.context[key] = undefined)}
													>
														<X />
														Cancel
													</Button>
													<Button
														variant="destructive"
														size="sm"
														onclick={() => handleSaveDelete('context', key, true)}
													>
														<Trash2 />
														Delete, with related clusters and users
													</Button>
													<Button
														variant="destructive"
														size="sm"
														onclick={() => handleSaveDelete('context', key)}
													>
														<Trash2 />
														Delete
													</Button>
												</div>
											</div>
										{:else}
											<div class="flex items-center justify-between">
												<div>
													<h4 class="font-medium">{context.name}</h4>
													<p class="text-sm text-muted-foreground">
														Cluster: {context.context.cluster || 'Unknown'}
														| User: {context.context.user || 'Unknown'}
													</p>
												</div>
												<div class="flex gap-1">
													<Button onclick={() => handleStartEdit('context', key)}>Edit</Button>
													<Button
														variant="destructive"
														onclick={() => handleStartDelete('context', key)}>Delete</Button
													>
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
				onclick={() => save()}
			>
				Save changes
			</Dialog.Close>
		</Dialog.Footer>
	</Dialog.Content>
</Dialog.Root>
