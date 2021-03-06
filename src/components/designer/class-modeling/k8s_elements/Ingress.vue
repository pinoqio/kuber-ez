<template>
    <div>
        <geometry-element
                selectable
                :movable="editMode"
                :resizable="editMode"
                connectable
                :deletable=editMode
                :id.sync="value.elementView.id"
                :x.sync="value.elementView.x"
                :y.sync="value.elementView.y"
                :width.sync="value.elementView.width"
                :height.sync="value.elementView.height"
                :angle.sync="value.elementView.angle"
                v-on:selectShape="selectedActivity"
                v-on:deSelectShape="deSelectedActivity"
                v-on:dblclick="showProperty"
                v-on:rotateShape="onRotateShape"
                v-on:labelChanged="onLabelChanged"
                v-on:addedToGroup="onAddedToGroup"
                v-on:removeShape="onRemoveShape(value)"
                :label.sync="name"
                :_style="{
                'label-angle':value.elementView.angle,
                'font-weight': 'bold','font-size': '16'
                }"
        >

            <!--v-on:dblclick="$refs['dialog'].open()"-->
            <geometry-rect
                    :_style="{
                        'fill-r': 1,
                        'fill-cx': .1,
                        'fill-cy': .1,
                        'stroke-width': 1.4,
                        'stroke': '#4caf50',
                        fill: '#4caf50',
                        'fill-opacity': 1,
                        r: '1',
                        'z-index': '998'
                    }"
            ></geometry-rect>

            <sub-elements>
                <!--title-->
                <text-element
                        :sub-width="'100%'"
                        :sub-height="30"
                        :sub-top="0"
                        :sub-left="0"
                        :text="'Ingress'">
                </text-element>
                <image-element
                        :image="imgSrc"
                        :sub-top="5"
                        :sub-left="5"
                        :sub-width="30"
                        :sub-height="30">
                </image-element>
            </sub-elements>
        </geometry-element>


        <property-panel
                v-if="openPanel"
                v-model="value"
                :img="imgSrc">
        </property-panel>
    </div>
</template>

<script>
    import Element from '../../modeling/Element'
    import PropertyPanel from './IngressPropertyPanel'

    export default {
        mixins: [Element],
        name: 'ingress',
        components: {
            "property-panel": PropertyPanel
        },
        props: {},
        computed: {
            defaultStyle() {
                return {}
            },
            className() {
                return 'Ingress'
            },
            imgSrc() {
                return `${ window.location.protocol + "//" + window.location.host}/static/image/symbol/kubernetes/ing.svg`
            },
            createNew(elementId, x, y, width, height) {
                return {
                    _type: this.className(),
                    name: '',
                    namespace:'',
                    elementView: {
                        '_type': this.className(),
                        'id': elementId,
                        'x': x,
                        'y': y,
                        'width': 150,
                        'height': 150,
                        'style': JSON.stringify({}),
                        'angle': 0,
                    },
                    object: {
                        "apiVersion": "extensions/v1beta1",
                        "kind": "Ingress",
                        "metadata": {
                            "name": "",
                        },
                        "spec": {
                            "rules": [
                                {
                                    "host": "insurance.infogra.io",
                                    "http": {
                                        "paths": [
                                            {
                                                "backend": {
                                                    "serviceName": "",
                                                    "servicePort": 80
                                                }
                                            }
                                        ]
                                    }
                                }
                            ]
                        }
                    },
                    outboundServices: [],
                    connectableType: ["Service"],
                    status: null,
                }
            },
            namespace: {
                get: function() {
                    return this.value.object.metadata.namespace
                },
                set: function (newVal){
                    this.value.object.metadata.namespace = newVal
                }
            },
            name() {
                try {
                    return this.value.object.metadata.name;
                } catch(e) {
                    return "Untitled";
                }
            },

            outboundServiceNames() {
                try {
                    var serviceNames = "";
                    
                    this.value.outboundServices.forEach(element => {
                        serviceNames += element.object.metadata.name + ":" + element.object.spec.ports[0].port +  ","
                    })

                    return serviceNames;

                } catch(e) {
                    return ""
                }
            },

            paths() {
                return this.value.object.spec.rules[0].http.paths
            },

        },
        data: function () {
            return {
                                
            };
        },
        created: function () {
            
        },
        mounted: function () {

            var me = this;

            this.$EventBus.$on(`${me.value.elementView.id}`, function (obj) {
                if(obj.state=="addRelation" && obj.element && obj.element.targetElement 
                    && obj.element.targetElement._type == "Service"){
                    
                    obj.element.targetElement.relationId = obj.element.relationView.id
                    me.value.outboundServices.push(obj.element.targetElement)
                }
                
                if(obj.state=="deleteRelation" && obj.element && obj.element.targetElement 
                    && obj.element.targetElement._type == "Service"){

                    me.value.outboundServices.splice(me.value.outboundServices.indexOf(obj.element.targetElement), 1);
                }

                if(obj.state == "get" && obj.element && obj.element.kind == me.value.object.kind) {
                    me.value.status = obj.element.status
                }

            })
            
        },
        watch: {
            "outboundServiceNames": function(names){

                this.value.object.spec.rules[0].http.paths = [];
                var me = this;
                this.value.outboundServices.forEach(element => {
                        me.value.object.spec.rules[0].http.paths.push(
                            {
                                "backend": {
                                    "serviceName": element.object.metadata.name,
                                    "servicePort": element.object.spec.ports[0].port
                                }
                            }
                        );
                    }
                );
            },

            paths: {
                deep: true,
                handler: function (newVal, oldVal) {
                    var me = this
                    if(newVal.length < oldVal.length) {
                        var index = me.getIndex(newVal, oldVal)

                        if (me.value.outboundServices[index]) {
                            me.deleteRelation(me.value.outboundServices[index].relationId)
                        }
                    }
                }
            }
        },

        methods: {
            getIndex(newArr, oldArr) {
                var index
                oldArr.some(function(item, idx) {
                    if (newArr[idx] == undefined) {
                        index = idx
                        return true
                    } else if (item.backend.serviceName != newArr[idx].backend.serviceName) {
                        index = idx
                        return true
                    }
                })
                return index
            },
        }
    }
</script>

<style>

</style>