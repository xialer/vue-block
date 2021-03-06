<script>
import {layout, calcCSS} from './algorithm';
import {mergeClassOrStyle, applyClass, applyStyle} from './vnode';

/**
 * Render VNodes.
 * if more than one vnode, then wrap it with div.
 * if empty, then return div.
 * @param {Function} h createElement function.
 * @param {Array.<VNode>} vnodes vnodes or null.
 * @param {*=} class class
 * @param {Array<Object>|Object=} style style.
 * @return {VNode} one vnode.
 */
function renderVNodes(h, vnodes, clazz, style) {
    if (vnodes && vnodes.length === 1) {
        // only one node
        const singleNode = vnodes[0];
        singleNode.data = singleNode.data || {};
        if (clazz) {
            singleNode.data.class = mergeClassOrStyle(singleNode.data.class, clazz);
        }
        if (style) {
            singleNode.data.style = mergeClassOrStyle(singleNode.data.style, style);
        }
        return singleNode;
    }

    // 2, 3, ...
    return <div class={clazz} style={style}>
        {vnodes || ''}
    </div>;
}

function renderDiv(isRoot, h, context, div) {
    // WOW, I love one piece. ^^
    const isOnepiece = !div.type;
    const spClazz = `block-pattern-${div.name}`;
    const clazz = {
        'block': true,
        'block-area': isOnepiece
    };
    clazz[spClazz] = isOnepiece;
    clazz[`block-${div.type}`] = !isOnepiece;
    calcCSS(div, context.props.rounder);
    const style = {
        width: div.csswidth || '',
        height: div.cssheight || '',
        minWidth: div.cssminwidth || '',
        minHeight: div.cssminheight || ''
    };

    if (div.split) {
        const userData = isRoot ? context.data : {};
        const vnodes = div.split.map(renderDiv.bind(null, false, h, context));
        return <div class={clazz} style={style} {...userData}>
            {isRoot ? renderAspectRatioWrapper(h, context, vnodes) : vnodes}
        </div>;
    }

    // is area
    const usedSlot = context.slots()[div.name];
    return renderVNodes(h, usedSlot, clazz, style);
}

function renderAspectRatioWrapper(h, context, vnodes) {
    const props = context.props;
    if (!props.aspectRatio) {
        return vnodes;
    }

    const ar = props.aspectRatio.trim().split(':').map((num) => parseInt(num, 10));
    const width = ar[0];
    const height = ar[1];
    const cssAR = `padding-bottom:${height * 100 / width}%`;
    return <div class="block-ar" style={cssAR}>
            <div class="block-ar-wrap">{vnodes}</div>
        </div>;
}

function renderDefault(h, context) {
    const slots = context.slots();
    const clazz = {};
    const vnodes = [];
    if (slots.left) {
        clazz['block-clear'] = true;
        applyClass(slots.left, 'block-left');
        vnodes.push.apply(vnodes, slots.left);
    }

    if (slots.center) {
        clazz['block-center'] = true;
        applyClass(slots.center, 'block-center');
        vnodes.push.apply(vnodes, slots.center);
    }

    if (slots.right) {
        clazz['block-clear'] = true;
        applyClass(slots.right.reverse(), 'block-right');
        vnodes.push.apply(vnodes, slots.right);
    }

    if (slots.middle) {
        //clazz['block-middle-container'] = true;
        vnodes.push(<div class="block-middle-container">
            <div class="block-middle">{slots.middle}</div>
        </div>);
    }

    if (slots.default) {
        vnodes.push.apply(vnodes, slots.default);
    }

    return <div class={clazz} {...context.data}>
            {renderAspectRatioWrapper(h, context, vnodes)}
        </div>;
}

export default {
    name: 'Block',
    functional: true,
    props: {
        rows: {
            type: String,
            default: 'auto'
        },
        cols: {
            type: String,
            default: 'auto'
        },
        pattern: String,
        rounder: {
            type: String,
            default: '100%'
        },
        // aspect-ratio
        aspectRatio: {
            type: String,
            validator(value) {
                return value.match(/^\s*\d+\s*:\s*\d+\s*$/);
            }
        }
    },
    render(h, context) {
        const props = context.props;
        let vnode;
        if (!props.pattern) {
            vnode = renderDefault(h, context);
        }
        else {
            const layouts = layout(props.pattern, props.rows, props.cols);
            vnode = renderDiv(true, h, context, layouts);
        }

        // jsx spread merge not support staticClass and staticStyle
        // https://github.com/vuejs/babel-plugin-transform-vue-jsx#difference-from-react-jsx
        // apply custom class and style.
        applyClass(vnode, context.data.staticClass);
        applyStyle(vnode, context.data.staticStyle);
        return vnode;
    }
}
</script>

<style lang="css">
.block {
    display: block;
    margin: 0;
    padding: 0;
}
.block-area {
    box-sizing: border-box;
}
.block-col.block-area,
.block-row.block-area {
    box-sizing: content-box;
}
.block-col {
    text-align: left;
}
.block-col > .block {
    display: inline-block;
    vertical-align: top;
}


.block-clear:before,
.block-clear:after {
    content: '';
    display: table;
    clear: both;
}
.block-left {
    float: left;
}
.block-right {
    float: right;
}


.block-center {
    text-align: center;
}
.block-middle > *,
.block-center > * {
    display: inline-block;
    vertical-align: top;
}
.block-middle-container {
    display: table;
    width: 100%;
    height: 100%;
}
.block-middle {
    display: table-cell;
    width: 100%;
    text-align: center;
    vertical-align: middle;
}

.block-ar,
.block-ar-wrap {
    margin: 0;
    padding: 0;
    border: 0;
    box-sizing: border-box;
}
.block-ar {
    position: relative;
    height: 0;
    overflow: visible;
}
.block-ar-wrap {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
}
</style>
