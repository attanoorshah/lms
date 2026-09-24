<template>
	<Dialog
		v-model:open="show"
		:title="isEdit ? __('Edit Member') : __('Add New Member')"
		size="lg"
		:actions="[
			{
				label: isEdit ? __('Save') : __('Add'),
				variant: 'solid',
				loading: submitting,
				onClick: ({ close }: any) => submit(close),
			},
		]"
	>
		<template #default>
			<div class="space-y-4">
				<FormControl
					v-model="member.email"
					:label="__('Email')"
					placeholder="jane@doe.com"
					type="email"
					:required="!isEdit"
					:disabled="isEdit"
					@keyup.enter="submit()"
				/>
				<div v-if="!isEdit" class="flex items-center gap-3">
					<FormControl
						v-model="member.first_name"
						:label="__('First Name')"
						placeholder="Jane"
						type="text"
						class="w-full"
					/>
					<FormControl
						v-model="member.last_name"
						:label="__('Last Name')"
						placeholder="Doe"
						type="text"
						class="w-full"
					/>
				</div>
				<Link
					v-model="member.ns_organization"
					doctype="Organization"
					:label="__('Organization')"
					:placeholder="__('Select organization')"
					:inlineCreate="true"
					inlineCreatePlaceholder="Organization name"
					:onCreate="onCreateOrganization"
				/>
				<Link
					v-model="member.ns_department"
					doctype="Department"
					:label="__('Department')"
					:placeholder="__('Select department')"
					:filters="{ organization: member.ns_organization }"
					:inlineCreate="true"
					inlineCreatePlaceholder="Department name"
					:onCreate="onCreateDepartment"
				/>
				<div class="flex flex-col gap-2">
					<div class="text-p-sm-medium text-ink-gray-7">
						{{ __('Roles') }}
					</div>
					<div class="grid md:grid-cols-2 gap-x-6 gap-y-3">
						<BooleanSwitch
							size="sm"
							:label="__('Student')"
							v-model="roles.lms_student"
						/>
						<BooleanSwitch
							size="sm"
							:label="__('Course Creator')"
							v-model="roles.course_creator"
						/>
						<BooleanSwitch
							size="sm"
							:label="__('Evaluator')"
							v-model="roles.batch_evaluator"
						/>
						<BooleanSwitch
							size="sm"
							:label="__('Moderator')"
							v-model="roles.moderator"
						/>
					</div>
				</div>
			</div>
		</template>
	</Dialog>
</template>

<script setup lang="ts">
import { call, Dialog, FormControl, toast } from 'frappe-ui'
import BooleanSwitch from '@/components/Controls/BooleanSwitch.vue'
import Link from '@/components/Controls/Link.vue'
import { computed, reactive, ref, watch } from 'vue'
import { cleanError } from '@/utils'

const show = defineModel<boolean>({ default: false })
const submitting = ref(false)

const props = defineProps<{
	defaultRoles?: string[]
	editMember?: {
		name: string
		full_name?: string
		roles?: string[]
		ns_organization?: string
		ns_department?: string
	} | null
}>()

const emit = defineEmits<{
	created: [user: any]
	updated: []
}>()

const isEdit = computed(() => !!props.editMember)

const ROLE_MAP: Record<string, string> = {
	moderator: 'Moderator',
	course_creator: 'Course Creator',
	batch_evaluator: 'Batch Evaluator',
	lms_student: 'LMS Student',
}

const member = reactive({
	email: '',
	first_name: '',
	last_name: '',
	ns_organization: '',
	ns_department: '',
})

const initialProfile = reactive({ ns_organization: '', ns_department: '' })

const roles = reactive({
	moderator: false,
	course_creator: false,
	batch_evaluator: false,
	lms_student: false,
})

const initialRoles = reactive({ ...roles })

const resetForm = () => {
	member.email = ''
	member.first_name = ''
	member.last_name = ''
	member.ns_organization = ''
	member.ns_department = ''
	applyDefaultRoles()
}

const applyDefaultRoles = () => {
	roles.moderator = props.defaultRoles?.includes('moderator') ?? false
	roles.course_creator = props.defaultRoles?.includes('course_creator') ?? false
	roles.batch_evaluator =
		props.defaultRoles?.includes('batch_evaluator') ?? false
	roles.lms_student = props.defaultRoles?.includes('lms_student') ?? false
}

const loadMember = () => {
	member.email = props.editMember?.name ?? ''
	member.first_name = ''
	member.last_name = ''
	member.ns_organization = props.editMember?.ns_organization ?? ''
	member.ns_department = props.editMember?.ns_department ?? ''
	initialProfile.ns_organization = member.ns_organization
	initialProfile.ns_department = member.ns_department
	const current = props.editMember?.roles ?? []
	for (const key of Object.keys(ROLE_MAP) as (keyof typeof roles)[]) {
		roles[key] = current.includes(ROLE_MAP[key])
		initialRoles[key] = roles[key]
	}
}

watch(show, (isOpen) => {
	if (!isOpen) return
	if (isEdit.value) loadMember()
	else resetForm()
})

const submit = (close?: () => void) => {
	if (submitting.value) return
	return isEdit.value ? saveRoles(close) : addMember(close)
}

const assignRoles = async (userEmail: string) => {
	const selectedRoles = Object.entries(roles).filter(([_, checked]) => checked)

	for (const [key, _] of selectedRoles) {
		await call('lms.lms.api.save_role', {
			user: userEmail,
			role: ROLE_MAP[key],
			value: 1,
		})
	}
}

const addMember = async (close?: () => void) => {
	if (!member.email?.trim()) {
		toast.error(__('Email is required'))
		return
	}

	submitting.value = true
	try {
		const user = await call('frappe.client.insert', {
			doc: {
				doctype: 'User',
				email: member.email.trim(),
				first_name: member.first_name.trim() || undefined,
				last_name: member.last_name.trim() || undefined,
				ns_organization: member.ns_organization || undefined,
				ns_department: member.ns_department || undefined,
			},
		})

		await assignRoles(user.name)

		toast.success(__('Member added successfully'))
		emit('created', user)
		resetForm()
		close?.()
	} catch (err: any) {
		toast.error(cleanError(err.messages?.[0]) || __('Unable to add member'))
	} finally {
		submitting.value = false
	}
}

const onCreateOrganization = (name: string, done: () => void) => {
	call('frappe.client.insert', {
		doc: { doctype: 'Organization', organization_name: name },
	}).then((data: any) => {
		member.ns_organization = data.name
		done()
	})
}

const onCreateDepartment = (name: string, done: () => void) => {
	if (!member.ns_organization) {
		toast.error(__('Select an Organization first'))
		return
	}
	call('frappe.client.insert', {
		doc: {
			doctype: 'Department',
			department_name: name,
			organization: member.ns_organization,
		},
	}).then((data: any) => {
		member.ns_department = data.name
		done()
	})
}

const saveProfile = async () => {
	if (!props.editMember?.name) return
	const changes: Record<string, string> = {}
	if (member.ns_organization !== initialProfile.ns_organization)
		changes.ns_organization = member.ns_organization
	if (member.ns_department !== initialProfile.ns_department)
		changes.ns_department = member.ns_department
	if (!Object.keys(changes).length) return

	await call('frappe.client.set_value', {
		doctype: 'User',
		name: props.editMember.name,
		fieldname: changes,
	})
	initialProfile.ns_organization = member.ns_organization
	initialProfile.ns_department = member.ns_department
}

const saveRoles = async (close?: () => void) => {
	if (!props.editMember?.name) return

	submitting.value = true
	try {
		await saveProfile()

		for (const key of Object.keys(ROLE_MAP) as (keyof typeof roles)[]) {
			if (roles[key] !== initialRoles[key]) {
				await call('lms.lms.api.save_role', {
					user: props.editMember.name,
					role: ROLE_MAP[key],
					value: roles[key] ? 1 : 0,
				})
			}
		}

		toast.success(__('Member updated'))
		emit('updated')
		close?.()
	} catch (err: any) {
		toast.error(cleanError(err.messages?.[0]) || __('Unable to update member'))
	} finally {
		submitting.value = false
	}
}
</script>
