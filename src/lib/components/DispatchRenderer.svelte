<script lang="ts" generics="U">
	import {getContext} from "svelte"
	import {
		type CoreActions,
		type Dispatch,
		type JsonFormsCellRendererRegistryEntry,
		type JsonFormsRendererRegistryEntry,
		type JsonFormsSubStates,
		type JsonSchema,
		mapStateToJsonFormsRendererProps,
		type StatePropsOfJsonFormsRenderer
	} from "@jsonforms/core"
	import {JsonFormsDispatchContextKey, JsonFormsSubStatesContextKey} from "../constants/context-keys.js"
	import maxBy from "lodash/maxBy.js"
	import UnknownRenderer from "./UnknownRenderer.svelte"

	interface Props<U> {
		schema: JsonSchema
		uischema: U
		path: string
		enabled?: boolean
		renderers?: JsonFormsRendererRegistryEntry[]
		cells?: JsonFormsCellRendererRegistryEntry[]
		config?: object
	}

	let allProps: Props<U> = $props()

	const jsonforms = getContext<JsonFormsSubStates>(JsonFormsSubStatesContextKey)
	const dispatch  = getContext<Dispatch<CoreActions>>(JsonFormsDispatchContextKey)

	if (!jsonforms || !dispatch) {
		throw new Error(
			"'jsonforms' or 'dispatch' not in the Svelte context. Are you within JSON Forms?"
		)
	}

	let rawProps: StatePropsOfJsonFormsRenderer = $derived(mapStateToJsonFormsRendererProps(
		{jsonforms},
		allProps
	))
	let rootSchema                              = $derived(rawProps.rootSchema)
	let renderer                                = $derived.by(() => {
		const {rootSchema: _rootSchema, ...rest} = rawProps
		return rest
	})
	let DeterminedRenderer                      = $derived.by(() => {
		const testerContext = {
			rootSchema: rootSchema,
			config:     allProps?.config,
		}
		const maxRenderer   = maxBy(renderer.renderers, (r) =>
			r.tester(renderer.uischema, renderer.schema, testerContext)
		)
		if (
			maxRenderer === undefined ||
			maxRenderer.tester(
				renderer.uischema,
				renderer.schema,
				testerContext
			) === -1
		) {
			return UnknownRenderer
		} else {
			return maxRenderer.renderer
		}
	})
</script>

<DeterminedRenderer {...allProps} {...renderer} />
