<script lang="ts" generics="U">
	import {getContext, onDestroy, onMount} from "svelte"
	import {JsonFormsDispatchContextKey, JsonFormsSubStatesContextKey} from "../constants/context-keys.js"
	import {
		type CoreActions,
		createId,
		type Dispatch,
		isControl,
		type JsonFormsCellRendererRegistryEntry,
		type JsonFormsRendererRegistryEntry,
		type JsonFormsSubStates,
		type JsonSchema,
		mapDispatchToControlProps,
		mapStateToDispatchCellProps,
		removeId
	} from "@jsonforms/core"
	import maxBy from "lodash/maxBy.js"
	import UnknownRenderer from "./UnknownRenderer.svelte"

	interface Props<U> {
		schema: JsonSchema
		uischema: U
		path: string
		enabled?: boolean
		renderers?: JsonFormsRendererRegistryEntry[]
		cells?: JsonFormsCellRendererRegistryEntry[]
		config?: Object
		[key: string]: any
	}

	let allProps: Props<U> = $props()
	// let {}                 = $derived(allProps)

	const jsonforms = getContext<JsonFormsSubStates>(JsonFormsSubStatesContextKey)
	const dispatch  = getContext<Dispatch<CoreActions>>(JsonFormsDispatchContextKey)

	if (!jsonforms || !dispatch) {
		throw new Error(
			"'jsonforms' or 'dispatch' not in the Svelte context. Are you within JSON Forms?"
		)
	}

	const dispatchMethods = mapDispatchToControlProps(dispatch)

	let id: string | undefined = $state(undefined)

	let control        = $derived({
		...allProps,
		...mapStateToDispatchCellProps({jsonforms}, allProps),
		id,
	})
	let DeterminedCell = $derived.by(() => {
		const testerContext = {
			rootSchema: control.rootSchema,
			config:     allProps.config,
		}
		const maxCell       = maxBy(control.cells, (r) =>
			r.tester(control.uischema, control.schema, testerContext)
		)
		if (
			maxCell === undefined ||
			maxCell.tester(control.uischema, control.schema, testerContext) === -1
		) {
			return UnknownRenderer
		} else {
			return maxCell.cell
		}
	})


	$effect(() => {
		if (isControl(control.uischema)) {
			if (id) {
				removeId(id)
			}
			id = createId(control.uischema.scope)
		}
	})

	onMount(() => {
		if (control.uischema.scope) {
			id = createId(control.uischema.scope)
		}
	})

	onDestroy(() => {
		if (id) {
			removeId(id)
			id = undefined
		}
	})
</script>

<DeterminedCell {...allProps} {...control} />
