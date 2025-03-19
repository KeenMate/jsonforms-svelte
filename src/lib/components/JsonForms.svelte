<script lang="ts">
	import DispatchRenderer from "$lib/components/DispatchRenderer.svelte"
	import {setContext, untrack} from "svelte"
	import {
		Actions,
		configReducer,
		type CoreActions,
		coreReducer,
		defaultMiddleware,
		type Dispatch,
		Generate,
		i18nReducer,
		type JsonFormsCellRendererRegistryEntry,
		type JsonFormsCore,
		type JsonFormsI18nState,
		type JsonFormsRendererRegistryEntry,
		type JsonFormsSubStates,
		type JsonFormsUISchemaRegistryEntry,
		type JsonSchema,
		type Middleware,
		type UISchemaElement,
		type ValidationMode
	} from "@jsonforms/core"
	import {JsonFormsDispatchContextKey, JsonFormsSubStatesContextKey} from "../constants/index.js"
	import {type Ajv, type ErrorObject} from "ajv"
	import {isObject} from "lodash"

	interface Props {
		data?: any
		schema?: JsonSchema
		uischema?: UISchemaElement
		renderers: JsonFormsRendererRegistryEntry[]
		cells?: JsonFormsCellRendererRegistryEntry[]
		config?: any
		readonly?: boolean
		uischemas?: JsonFormsUISchemaRegistryEntry[]
		validationMode?: ValidationMode
		ajv?: Ajv
		i18n?: JsonFormsI18nState
		additionalErrors?: ErrorObject[]
		middleware?: Middleware
		onChange?: (changeData: any) => void
	}

	let {
		    data             = undefined,
		    schema           = undefined,
		    uischema         = undefined,
		    renderers,
		    cells            = [],
		    config           = undefined,
		    readonly         = undefined,
		    uischemas        = [],
		    validationMode   = "ValidateAndShow",
		    ajv              = undefined,
		    i18n             = undefined,
		    additionalErrors = [],
		    middleware       = defaultMiddleware,
		    onChange         = undefined
	    }: Props = $props()

	let dataToUse               = $state()
	let generatorData           = $state(isObject(dataToUse) ? dataToUse : {})
	let schemaToUse: JsonSchema = $state(schema ?? Generate.jsonSchema(generatorData))
	let uischemaToUse           = $state(uischema ?? Generate.uiSchema(schemaToUse, undefined, undefined, schemaToUse))
	let jsonforms               = $state({
		core:      initCore(),
		config:    configReducer(undefined, Actions.setConfig(config)),
		i18n:      i18nReducer(
			i18n,
			Actions.updateI18n(
				i18n?.locale,
				i18n?.translate,
				i18n?.translateError
			)
		),
		renderers: renderers,
		cells:     cells,
		uischemas: uischemas,
		readonly:  readonly,
	})

	setContext<JsonFormsSubStates>(JsonFormsSubStatesContextKey, jsonforms)
	setContext<Dispatch<CoreActions>>(JsonFormsDispatchContextKey, dispatch)

	$effect(() => {
		const generatorData = isObject(untrack(() => data)) ? untrack(() => data) : {}
		schemaToUse         = schema ?? Generate.jsonSchema(generatorData)
		if (!untrack(() => uischema)) {
			uischemaToUse = Generate.uiSchema(
				untrack(() => schemaToUse),
				undefined,
				undefined,
				untrack(() => schemaToUse)
			)
		}
	})
	$effect(() => {
		uischemaToUse =
			uischema ??
			Generate.uiSchema(
				untrack(() => schemaToUse),
				undefined,
				undefined,
				untrack(() => schemaToUse)
			)
	})
	$effect(() => {
		dataToUse = data
	})
	$effect(() => {
		jsonforms.renderers = renderers
	})
	$effect(() => {
		jsonforms.cells = cells
	})
	$effect(() => {
		jsonforms.uischemas = uischemas
	})
	$effect(() => {
		jsonforms.config = configReducer(
			undefined,
			Actions.setConfig(config)
		)
	})
	$effect(() => {
		jsonforms.readonly = readonly
	})
	$effect(() => {
		jsonforms.core = untrack(() => middleware)(
			untrack(() => jsonforms.core) as JsonFormsCore,
			Actions.updateCore(
				dataToUse,
				schemaToUse,
				uischemaToUse,
				{
					validationMode:   validationMode,
					ajv:              ajv,
					additionalErrors: additionalErrors,
				}
			),
			coreReducer
		)
	})

	function initCore(): JsonFormsCore {
		const initialCore = {
			data:     dataToUse,
			schema:   schemaToUse,
			uischema: uischemaToUse,
		}
		const core        = middleware(
			initialCore,
			Actions.init(dataToUse, schemaToUse, uischemaToUse, {
				validationMode:   validationMode,
				ajv:              ajv,
				additionalErrors: additionalErrors,
			}),
			coreReducer
		)
		return core
	}

	function dispatch(action: CoreActions) {
		jsonforms.core = middleware(
			jsonforms.core as JsonFormsCore,
			action,
			coreReducer
		)

		if (["jsonforms/UPDATE", "jsonforms/UPDATE_ERRORS"].includes(action.type)) {
			onChange?.({
				data:   jsonforms.core.data,
				errors: jsonforms.core.errors,
			})
		}
	}
</script>

<DispatchRenderer
	schema={jsonforms.core.schema}
	uischema={jsonforms.core.uischema}
	path={""}
/>
