<template>
    <div class="icons" :class="iconsClasses" ref="iconsWrapper">
        <div ref="icons">
            <v-icon ref="lockIcon" color="secondary" small>{{lockIconIcon}}</v-icon>
        </div>
    </div>
</template>
<script lang="ts">
import Component from 'vue-class-component';
import {Mixins, Prop, Ref, Watch} from 'vue-property-decorator';
import { Node } from '../../graph/Node';
import Cytoscape, {ElementDefinition, NodeDataDefinition} from "cytoscape";
import Vue from 'vue';
import {NodeView} from '../../graph/NodeView';
import clone from "clone";
import GraphElementNodeMixin from "./GraphElementNodeMixin";

/**
 * This is Vue component representing single node in graph. When a new node is loaded,
 * the Vuex automatically creates a new instance of this class which registers node
 * in the Cytoscape instance. Also any changes of the Node's data are automatically
 * updated in the Cytoscape instance.
 *
 * This component, except the node registration, also renders some HTML. For now, it
 * renders lock icon used for marking node as locked.
 */
@Component
export default class GraphElementNode extends Mixins(GraphElementNodeMixin) {
    /**
     * Node's data passed by parent
     */
    @Prop({type: Object as () => Node}) node: Node;

    /**
     * @inheritDoc
     */
    protected registerElement(): void {
        let position = this.node.onMountPosition ? {x: this.node.onMountPosition[0], y: this.node.onMountPosition[1]} : {x: 0, y: 0};

        // All parameters here must correspond to functions trigger by watchers
        // The objects are considered owned by Cytoscape therefore are copied to remove Vue observers
        this.element = <Cytoscape.NodeSingular>this.cy.add({
            group: 'nodes',
            // label: Fixes Cytoscape bug when there is no clickable bounding box when node has [width: label] and previous label was empty
            data: { label: "-", ...clone(this.node.lastPreview), id: this.node.IRI } as NodeDataDefinition,
            classes: this.classList as unknown as string,
            position,
        } as ElementDefinition);

        this.lockedForLayoutsChangedPopper();
    }

    //#region Methods for Popper manipulation and lock icon
    protected popperIsActive: boolean;
    @Ref() protected readonly iconsWrapper !: HTMLDivElement;
    @Ref() protected readonly icons !: HTMLDivElement;
    @Ref() protected readonly lockIcon !: Vue;

    @Watch('lockedForLayoutsActive')
    private lockedForLayoutsChangedPopper() {
        this.popperIsActive = this.lockedForLayoutsActive;
    }
    //#endregion Methods for Popper manipulation and lock icon

    /**
     * Method called by ancestor component GraphArea when doubleclick is registered
     */
    async onDoubleClicked() {
        let view: NodeView;

        if (!this.node?.currentView.IRI) {
            // Currently nodes obtained by expansion have view, but it does not contain IRI
            view = await this.node.getDefaultView();
        } else {
            view = this.node.currentView;
        }

        if (!view) return;

        await this.areaManipulator.expandNode(view);
    }

    /**
     * When node changes the current view and preview is not fetched yet,
     * this function is called with empty preview and therefore the old
     * values saved in Cy instance are not overwritten.
     */
    @Watch('node.lastPreview', { deep: true })
    @Watch('classList')
    private updatePreview() {
        // By doing this we achieve that the current preview remains if there is no other
        if (this.node.lastPreview !== null) {
            // Reset all data

            // The reason why this.element.removeData() is not used is because of a bug in Cytoscape library.
            // The data field label Preview must not be empty otherwise the node become un clickable when [width: label] style
            // is used
            let emptyData: any = {};
            if (this.element) {
                for (let id in this.element.data()) {
                    emptyData[id] = undefined;
                }
            }
            this.element?.data({
                ...emptyData,
                label: "-",
                ...clone(this.node.lastPreview),
                id: this.node.IRI
            });
        }

        // @ts-ignore bad types
        this.element?.classes(this.classList);
    }

    /**
     * Functions return ready class list which can be used to pass to cytoscape
     */
    protected get classList(): string[] {
        return ["__node", ...this._getClassList(), ...clone(this.node.lastPreview?.classes ?? [])];
    }

}
</script>
<style scoped lang="scss" src="./icons.scss" />
