<template>
	<div v-bem>
		<div v-bem:container>
			<!-- Preview -->
			<div
				v-bem:preview-wrapper="{ mode: previewMode }"
				@click="(previewMode === 'single') && (previewActive = !previewActive)"
			>
				<!-- Single -->
				<template v-if="isSinglePreview">
					<component :is="previewTransition" appear v-bind="cOptions">
						<div v-if="previewActive" v-bem:preview-single>{{ cTransitionLabel }}</div>
					</component>
				</template>

				<!-- Group -->
				<template v-if="isGroupPreview">
					<component
						:is="previewTransition"
						v-bem:preview-group
						v-bind="cOptions"
						group
					>
						<div
							v-for="(item, index) in previewItems"
							v-bem:preview-group-item
							:key="item.hash"
							:style="item.styles"
							@click.self="removeItem($event, index);"
						>
							<div v-bem:preview-group-item-add>+</div>
						</div>
					</component>
				</template>
			</div>

			<!-- Controls -->
			<div v-bem:controls-wrapper>
				<div v-bem:controls>
					<!-- Transition -->
					<el-select v-bem:controls-selector v-model="previewTransition">
						<el-option v-for="option in $options.TRANSITIONS_LIST" :key="option.value" :value="option.value">
							{{ option.label }}
						</el-option>
					</el-select>
					<!-- Type @TODO: -->
					<!-- <el-radio-group v-bem:controls-type v-model="previewMode">
						<el-radio-button label="single">Single</el-radio-button>
						<el-radio-button label="group">Group</el-radio-button>
						</el-radio-group> -->
					<!-- Separate values -->
					<el-checkbox
						v-bem:controls-separate
						v-model="isSeparated"
						label="Separate values for enter and leave animation"
					/>
				</div>
			</div>

			<el-divider />

			<!-- Controls -->
			<div v-bem:controls>
				<!-- Options -->
				<transition-slide v-bem:controls-options v-bind="{ group: true, tag: 'div' }">
					<!-- Duration -->
					<prop-control key="prop-duration" v-model="duration" v-bind="{ isSeparated }" label="Duration, ms" />
					<!-- Delay -->
					<prop-control key="prop-delay" v-model="delay" v-bind="{ isSeparated }" label="Delay, ms" />
					<!-- Easing -->
					<prop-control
						key="prop-easing"
						v-model="easing"
						v-bind="{ isSeparated }"
						label="Easing, CSS transition function"
						type="autocomplete"
						:options="$options.easingAutocompleteOptions"
					/>

					<!-- Axis (expand && scale) -->
					<prop-control
						v-if="['transition-expand', 'transition-scale'].includes(previewTransition)"
						key="prop-axis"
						v-model="axis"
						v-bind="{ isSeparated }"
						label="Axis"
						type="select"
						:options="previewTransition === 'transition-scale' ? $options.scaleAxisOptions : $options.axisOptions"
					/>

					<!-- Origin (scale) -->
					<prop-control
						v-if="previewTransition === 'transition-scale'"
						key="prop-origin"
						v-model="origin"
						v-bind="{ isSeparated }"
						label="Origin"
						type="text"
					/>

					<!-- Offset (slide) -->
					<prop-control
						v-if="previewTransition === 'transition-slide'"
						key="prop-origin"
						v-model="offset"
						v-bind="{ isSeparated }"
						label="Offset"
						type="double-text"
					/>

					<!-- Move duration -->
					<prop-control
						v-if="previewMode === 'group'"
						key="prop-move-duration"
						v-model="moveDuration"
						label="Move duration, ms"
					/>
				</transition-slide>
			</div>
		</div>
	</div>
</template>

<script>
	import { randomString, randomInteger, isArray } from '@morev/helpers';
	import { TransitionFade, TransitionExpand, TransitionSlide, TransitionScale } from '../../../src/index.js';
	import * as defaults from '../../../src/utility/defaults/defaults.js';
	import OptionsGroup from '../options-group/options-group.vue';
	import PropControl from '../prop-control/prop-control.vue';

	const TRANSITIONS_LIST = [
		{ value: 'transition-fade', label: '<transition-fade />' },
		{ value: 'transition-expand', label: '<transition-expand />' },
		{ value: 'transition-slide', label: '<transition-slide />' },
		{ value: 'transition-scale', label: '<transition-scale />' },
	];

	const easingAutocompleteOptions = ['ease', 'ease-in', 'ease-in-out', 'ease-out', 'linear'];
	const axisOptions = ['x', 'y'];
	const scaleAxisOptions = ['both', 'x', 'y'];

	const withEnterLeave = (v) => (isArray(v) ? { enter: v[0], leave: v[1] } : v);

	export default {
		TRANSITIONS_LIST,
		easingAutocompleteOptions,
		axisOptions,
		scaleAxisOptions,
		name: 'transition-proxy',
		components: {
			OptionsGroup,
			TransitionFade,
			TransitionExpand,
			TransitionSlide,
			TransitionScale,
			PropControl,
		},
		data: () => ({
			previewTransition: 'transition-slide',
			previewMode: 'single',
			previewActive: true,
			previewItems: [],
			isSeparated: false,

			duration: defaults.transitionDuration,
			delay: defaults.transitionDelay,
			easing: defaults.transitionEasing,
			axis: defaults.expandAxis,
			origin: defaults.scaleOrigin,
			offset: defaults.slideOffset,
			moveDuration: defaults.moveDuration,
		}),
		computed: {
			isSinglePreview() {
				return this.previewMode === 'single';
			},
			isGroupPreview() {
				return this.previewMode === 'group';
			},
			cTransitionLabel() {
				return TRANSITIONS_LIST.find(t => t.value === this.previewTransition).label;
			},
			cOptions() {
				const duration = withEnterLeave(this.duration);
				const easing = withEnterLeave(this.easing);
				const delay = withEnterLeave(this.delay);
				const axis = withEnterLeave(this.axis);
				const origin = withEnterLeave(this.origin);
				const offset = Array.isArray(this.offset[0])
					? { enter: this.offset[0], leave: this.offset[1] }
					: this.offset;

				const moveDuration = this.optionMoveDuration;

				return {
					duration,
					easing,
					delay,
					offset,
					axis,
					origin,
					moveDuration,
				};
			},
		},
		watch: {
			cTransition() {
				this.switchPreview();
			},
			cGroup() {
				this.switchPreview();
			},
		},
		methods: {
			switchPreview() {
				const delay = (this.previewActive ? this.optionDurationOut : 0) + 20;
				this.previewActive = false;

				setTimeout(() => {
					this.previewTransition = this.cTransition;
					this.previewMode = this.previewMode === 'single' ? 'group' : 'single';
					this.previewActive = true;
				}, delay);
			},
			addItem() {
				const index = randomInteger(0, this.previewItems.length);
				this.previewItems.splice(index, 0, {
					hash: randomString(),
					styles: {
						backgroundColor: `rgba(${[
							randomInteger(0, 255),
							randomInteger(0, 255),
							randomInteger(0, 255),
						]})`,
					},
				});
			},
			removeItem(e, index) {
				const random = randomInteger(0, this.previewItems.length - 1);
				this.previewItems.splice(index ?? random, 1);
			},
		},
		created() {
			for (let i = 0; i < 7; i++) {
				this.addItem();
			}
		},
	};
</script>

<style lang="scss" src="./transition-proxy.scss"></style>
