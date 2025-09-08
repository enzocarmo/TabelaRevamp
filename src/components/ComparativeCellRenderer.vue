<template>
    <div class="comparative-cell">
        <span :class="{ 'negative': isNegative, 'positive': isPositive, 'zero': isZero }">
            {{ formattedValue }}
            <PhCaretDown v-if="isNegative && hasValue" :size="20" weight="fill" class="icon" />
            <PhCaretUp v-if="isPositive && hasValue" :size="20" weight="fill" class="icon" />
            <PhMinus v-if="isZero && hasValue" :size="20" weight="bold" class="icon" />
        </span>
    </div>
</template>

<script>
import { PhCaretUp, PhCaretDown, PhMinus } from '@phosphor-icons/vue'

export default {
    components: {
        PhCaretUp,
        PhCaretDown,
        PhMinus
    },
    name: "ComparativeCellRenderer",
    props: {
        params: {
            type: Object,
            required: true,
        },
    },
    computed: {
        // NOVA LÓGICA: Usa o valor formatado se disponível para comparação
        valueForComparison() {
            // Se há valueFormatted (valor customizado), extrair o número dele
            if (this.params.valueFormatted && this.params.valueFormatted !== this.params.value) {
                // Remove símbolos de moeda, porcentagem e formatação
                const cleanValue = String(this.params.valueFormatted)
                    .replace(/[R$\s%]/g, '') // Remove R$, espaços e %
                    .replace(/\./g, '') // Remove pontos de milhar
                    .replace(',', '.'); // Troca vírgula por ponto decimal

                const numericValue = parseFloat(cleanValue);
                return !isNaN(numericValue) ? numericValue : this.params.value;
            }
            // Caso contrário, usa o valor original
            return this.params.value;
        },
        hasValue() {
            const compareValue = this.valueForComparison;
            return compareValue !== null && compareValue !== undefined && !isNaN(compareValue);
        },
        isNegative() {
            return this.hasValue && this.valueForComparison < 0;
        },
        isZero() {
            return this.hasValue && this.valueForComparison === 0;
        },
        isPositive() {
            return this.hasValue && this.valueForComparison > 0;
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

        &.zero {
            color: #140F10;
        }

        .icon {
            padding-left: 4px;
        }
    }
}
</style>