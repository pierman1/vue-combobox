<template>
	<div class="select-wrapper">
		<div class="select-custom">
			<label id="select-label" class="select-label sr-only"
				>{{ label }}</label
			>
			<div class="select" :class="{ open: open }">
				<div
					:aria-controls="`listbox-${_uid}`"
					:aria-expanded="open"
					aria-haspopup="listbox"
					aria-labelledby="select-label"
					id="select"
					class="select-input"
					role="selectbox"
					tabindex="0"
					ref="select"
					@blur="onBlur"
					@keydown="onKeyDown"
					@click="updateMenuState(!open, false)"
					:aria-activedescendant="`combo-${activeIndex}`"
				>
					{{ options?.[selectedOptionIndex]?.label }}
				</div>
				<div
					class="select-menu"
					role="listbox"
					:id="`listbox-${_uid}`"
					aria-labelledby="select-label"
					tabindex="-1"
					ref="select-list"
				>
					<div
						v-for="(option, index) in options"
						role="option"
						class="select-option"
						:class="{ 'option-current': index === activeIndex }"
						:id="`combo-${index}`"
						:key="`combo-${index}`"
						:ref="`combo-${index}`"
						:aria-selected="index === selectedOptionIndex"
						@mousedown="onOptionMouseDown()"
						@click="onOptionClick($event, index)"
					>
						{{ option.label }}
					</div>
				</div>
			</div>
		</div>

		<div class="select-native">
			<label for="select"></label>
			<select class="select" v-model="selectedOptionIndex" name="select" @select="$emit($event.value)">
				<option
					v-for="(option, index) in options"
					:key="`option_${index}`"
					:value="index">{{option.label}}</option>
			</select>
		</div>
	</div>
</template>

<script>
const SelectActions = {
	Close: 0,
	CloseSelect: 1,
	First: 2,
	Last: 3,
	Next: 4,
	Open: 5,
	PageDown: 6,
	PageUp: 7,
	Previous: 8,
	Select: 9,
	Type: 10,
};

export default {
	name: 'SelectBox',
	props: {
		label: {
			type: String,
		},
		options: {
			type: Array,
		}
	},
	data() {
		return {
			open: false,
			preSelectedIndex: 1,
			selectedOptionIndex: null,
			ariaExpanded: false,
			activeIndex: null,
			currentIndex: null,
			ignoreBlur: false,
			searchTimeout: null,
		};
	},
	mounted() {
		// if a selectedIndex is given we set that option to be selected
		if (this.preSelectedIndex) {
			this.selectedOptionIndex = this.preSelectedIndex;
		}
	},
	methods: {
		updateMenuState(open, callFocus = true) {
			if (this.open === open) {
				return;
			}

			// update state (this also opens the select)
			this.open = open;

			// move focus back to the select, if needed
			callFocus && this.$refs['select'].focus();
		},
		onOptionMouseDown() {
			// Clicking an option will cause a blur event,
			// but we don't want to perform the default keyboard blur action
			this.ignoreBlur = true;
		},
		onBlur() {
			// do not do blur action if ignoreBlur flag has been set
			if (this.ignoreBlur) {
				this.ignoreBlur = false;
				return;
			}

			// select current option and close
			if (this.open) {
				this.selectOption(this.activeIndex || this.selectedOptionIndex);
				this.updateMenuState(false, false);
			}
		},
		onOptionChange(index) {
			// update state
			this.activeIndex = index;
			// TODO: ensure the new option is in view

			const option = this.$refs[`combo-${index}`];

			// check if list is scrollable
			if (this.isScrollable(this.$refs['select-list'])) {
				// ensure the new option is in view
				this.maintainScrollVisibility(option[0], this.$refs['select-list']);
			}
		},
		maintainScrollVisibility(activeElement, scrollParent) {
			const { offsetHeight, offsetTop } = activeElement;
			const { offsetHeight: parentOffsetHeight, scrollTop } = scrollParent;

			const isAbove = offsetTop < scrollTop;
			const isBelow = offsetTop + offsetHeight > scrollTop + parentOffsetHeight;

			if (isAbove) {
				scrollParent.scrollTo(0, offsetTop);
			} else if (isBelow) {
				scrollParent.scrollTo(0, offsetTop - parentOffsetHeight + offsetHeight);
			}
		},
		isScrollable(element) {
			return element && element.clientHeight < element.scrollHeight;
		},
		onKeyDown(event) {
			const { key } = event;
			const max = this.options.length - 1;

			const action = this.getActionFromKey(event, this.open);

			switch (action) {
				case SelectActions.Last:
				case SelectActions.First:
					this.updateMenuState(true);
				// intentional fallthrough
				case SelectActions.Next:
				case SelectActions.Previous:
				case SelectActions.PageUp:
				case SelectActions.PageDown:
					event.preventDefault();
					return this.onOptionChange(
						this.getUpdatedIndex(this.activeIndex, max, action)
					);
				case SelectActions.CloseSelect:
					event.preventDefault();
					this.selectOption(this.activeIndex);
				// intentional fallthrough
				case SelectActions.Close:
					event.preventDefault();
					return this.updateMenuState(false);
				case SelectActions.Type:
					return this.onComboType(key);
				case SelectActions.Open:
					event.preventDefault();
					return this.updateMenuState(true);
			}
		},
		selectOption(index) {
			// update state
			this.selectedOptionIndex = index;
		},
		onComboType(letter) {
			// open the listbox if it is closed
			this.updateMenuState(true);
			// find the index of the first matching option
			const searchString = this.getSearchString(letter);
			const searchIndex = this.getIndexByLetter(
				searchString,
				this.activeIndex + 1
			);

			// if a match was found, go to it
			if (searchIndex >= 0) {
				this.onOptionChange(searchIndex);
			}
			// if no matches, clear the timeout and search string
			else {
				window.clearTimeout(this.searchTimeout);
				this.searchString = '';
			}
		},
		getSearchString(char) {
			// reset typing timeout and start new timeout
			// this allows us to make multiple-letter matches, like a native select
			if (typeof this.searchTimeout === 'number') {
				window.clearTimeout(this.searchTimeout);
			}

			this.searchTimeout = window.setTimeout(() => {
				this.searchString = '';
			}, 500);

			// add most recent letter to saved search string
			this.searchString += char;
			return this.searchString;
		},
		filterOptions(filter, options = [], exclude = []) {
			return options.filter((option) => {
				const matches =
					option.label.toLowerCase().indexOf(filter.toLowerCase()) === 0;
				return matches && exclude.indexOf(option.label) < 0;
			});
		},
		getIndexByLetter(filter, startIndex = 0) {
			const orderedOptions = [
				...this.options.slice(startIndex),
				...this.options.slice(0, startIndex),
			];
			const firstMatch = this.filterOptions(filter, orderedOptions)[0];
			const allSameLetter = (array) =>
				array.every((letter) => letter === array[0]);

			// first check if there is an exact match for the typed string
			if (firstMatch) {
				return this.options.indexOf(firstMatch);
			}

			// if the same letter is being repeated, cycle through first-letter matches
			else if (allSameLetter(filter.split(''))) {
				const matches = this.filterOptions(filter[0], orderedOptions);
				return this.options.indexOf(matches[0]);
			}

			// if no matches, return -1
			else {
				return -1;
			}
		},
		getActionFromKey(event, menuOpen) {
			const { key, altKey, ctrlKey, metaKey } = event;
			const openKeys = ['ArrowDown', 'ArrowUp', 'Enter', ' ']; // all keys that will do the default open action
			// handle opening when closed
			if (!menuOpen && openKeys.includes(key)) {
				return SelectActions.Open;
			}

			// home and end move the selected option when open or closed
			if (key === 'Home') {
				return SelectActions.First;
			}
			if (key === 'End') {
				return SelectActions.Last;
			}

			// handle typing characters when open or closed
			if (
				key === 'Backspace' ||
				key === 'Clear' ||
				(key.length === 1 && key !== ' ' && !altKey && !ctrlKey && !metaKey)
			) {
				return SelectActions.Type;
			}

			// handle keys when open
			if (menuOpen) {
				if (key === 'ArrowUp' && altKey) {
					return SelectActions.CloseSelect;
				} else if (key === 'ArrowDown' && !altKey) {
					return SelectActions.Next;
				} else if (key === 'ArrowUp') {
					return SelectActions.Previous;
				} else if (key === 'PageUp') {
					return SelectActions.PageUp;
				} else if (key === 'PageDown') {
					return SelectActions.PageDown;
				} else if (key === 'Escape') {
					return SelectActions.Close;
				} else if (key === 'Enter' || key === ' ') {
					return SelectActions.CloseSelect;
				}
			}
		},
		getUpdatedIndex(currentIndex, maxIndex, action) {
			const pageSize = 2; // used for pageup/pagedown

			switch (action) {
				case SelectActions.First:
					return 0;
				case SelectActions.Last:
					return maxIndex;
				case SelectActions.Previous:
					return Math.max(0, currentIndex - 1);
				case SelectActions.Next:
					return Math.min(maxIndex, currentIndex + 1);
				case SelectActions.PageUp:
					return Math.max(0, currentIndex - pageSize);
				case SelectActions.PageDown:
					return Math.min(maxIndex, currentIndex + pageSize);
				default:
					return currentIndex;
			}
		},
		onOptionClick(event, index) {
			event.preventDefault();
			event.stopPropagation();
			this.onOptionChange(index);
			this.selectOption(index);
			this.updateMenuState(false, true);
		},
	},
};
</script>

<style scoped lang="scss">
[aria-selected='true'] {
	background: $color-grey-20;
}

.select *,
.select *::before,
.select *::after {
	box-sizing: border-box;
}

.select {
	display: block;
	min-width: inherit;
	position: relative;

	&.open {
		&::after {
			transform: translateY(-50%) rotate(-180deg);
		}
	}
}

.select::after {
	content: '';
	display: block;
	height: 9px;
	pointer-events: none;
	position: absolute;
	right: 16px;
	width: 9px;
	top: 50%;
	transform: translateY(-50%);
	background-size: 100%;
	background-repeat: no-repeat;
	background-position: center;
	background-image: url('~/assets/icons/chevron-down.svg');
	transition: 125ms transform;
}

.select-input {
	border: 1px solid #797876;
	border-radius: 12px;
	display: block;
	font-size: 1em;
	min-height: calc(1.4em + 26px);
	padding: 12px 32px 12px 16px;
	text-align: left;
	width: 100%;
	color: #3c4c4f;
}

.open .select-input {
	border-radius: 12px 12px 0 0;
}

.select-input:focus {
	outline-offset: -3px;
	outline: 2px solid $color-focus;
}

.select-label {
	display: block;
	font-size: 20px;
	font-weight: 100;
	margin-bottom: 0.25em;
}

.select-menu {
	background-color: $color-white;
	border: 1px solid #797876;
	border-radius: 0 0 12px 12px;
	display: none;
	max-height: 200px;
	overflow-y: scroll;
	left: 0;
	position: absolute;
	top: 100%;
	width: 100%;
	z-index: 100;
}

.open .select-menu {
	display: block;
	border-top: 0px;
}

.select-option {
	padding: 12px 12px 12px 16px;
	cursor: pointer;
}

.select-option:hover {
	background-color: rgba(209, 209, 209, 0.1);
}

.select-option.option-current {
	outline: 3px solid $color-focus;
	outline-offset: -3px;
}

.select-option[aria-selected='true'] {
	padding-right: 30px;
	position: relative;
}

.select-option[aria-selected='true']::after {
	content: '';
	height: 12px;
	position: absolute;
	right: 15px;
	top: 50%;
	transform: translate(0, -50%);
	width: 12px;
	background-image: url('~/assets/icons/checkmark.svg');
	background-size: 100%;
	background-repeat: no-repeat;
	background-position: center;
}

/*
	SELECT-NATIVE STYLES
*/
.select-native {
	border: 1px solid #797876;
  border-radius: 12px;
  display: block;
  font-size: 1em;
  min-height: calc(1.4em + 26px);
  text-align: left;
  width: 100%;
	color: #3C4C4F;
	position: relative;

	&::after {
		content: "";
		display: block;
		height: 9px;
		pointer-events: none;
		position: absolute;
		right: 16px;
		width: 9px;
		top: 50%;
		transform: translateY(-50%);
		background-size: 100%;
		background-repeat: no-repeat;
		background-position: center;
		background-image: url('~/assets/icons/chevron-down.svg');
		transition: 125ms transform;
	}

	select {
		padding: 12px 32px 12px 16px;
		border-radius: 12px;
		border: none;
		width: 100%;
		-moz-appearance:none; /* Firefox */
    -webkit-appearance:none; /* Safari and Chrome */
    appearance:none;
	}
}

/*
	SHOW NATIVE SELECT ON MOBILE & CUSTOM SELECT ON WIDE
*/
.select-wrapper {
	.select-custom {
		display: none;
	}

	@include media('>wide') {
		.select-native {
			display: none;
		}

		.select-custom {
			display: block;
		}
	}
}
</style>
