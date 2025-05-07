<template>
    <div class="comparative-cell">
        <span :class="{ 'negative': isNegative, 'positive': !isNegative && hasValue }">
            {{ formattedValue }}
            <PhCaretDown v-if="isNegative && hasValue" :size="20" weight="fill" class="icon" />
            <PhCaretUp v-if="!isNegative && hasValue" :size="20" weight="fill" class="icon" />
        </span>
    </div>
</template>

<script>
import { PhCaretUp, PhCaretDown } from '@phosphor-icons/vue'

export default {
    components: {
        PhCaretUp,
        PhCaretDown
    },
    name: "ComparativeCellRenderer",
    props: {
        params: {
            type: Object,
            required: true,
        },
    },
    computed: {
        hasValue() {
            return this.params.value !== null && this.params.value !== undefined;
        },
        isNegative() {
            // Parse string or number values
            const value = typeof this.params.value === 'string'
                ? parseFloat(this.params.value)
                : this.params.value;
            return this.hasValue && value < 0;
        },
        formattedValue() {
            return this.params.valueFormatted || this.params.value;
        }
    }
};
</script>

<style scoped lang="scss">
.comparative-cell {
    height: 100%;
    display: flex;
    align-items: center;

    span {
        display: flex;
        align-items: center;

        &.negative {
            color: #F8471C;
        }

        &.positive {
            color: #27be52;
        }

        .icon {
            padding-left: 4px;
        }
    }
}
</style>