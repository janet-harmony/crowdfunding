<template>
    <v-app class="grey lighten-3">
        <v-content>
            <v-container>
                <v-layout
                        text-xs-center
                        wrap
                >
                    <v-flex mb-4>
                        <h1 class="display-2 font-weight-bold mb-3">
                            [Untitled Crowdfunding dApp]
                        </h1>
                        <p class="subheading font-weight-regular">
                            Decentralized Crowdfunding on Harmony
                        </p>
                    </v-flex>
                </v-layout>

                <v-layout row justify-center>
                    <v-dialog v-model="startProjectDialog" max-width="600px" persistent>
                        <v-btn slot="activator" color="primary" dark>Start a Project</v-btn>
                        <v-card>
                            <v-card-title>
                                <span class="headline font-weight-bold mt-2 ml-4">Bring your project to life</span>
                            </v-card-title>
                            <v-card-text class="pt-0">
                                <v-container class="pt-0" grid-list-md>
                                    <v-layout wrap>
                                        <v-flex xs12>
                                            <v-text-field
                                                    label="Title"
                                                    persistent-hint
                                                    v-model="newProject.title">
                                            </v-text-field>
                                        </v-flex>
                                        <v-flex xs12>
                                            <v-textarea
                                                    label="Description"
                                                    persistent-hint
                                                    v-model="newProject.description">
                                            </v-textarea>
                                        </v-flex>
                                        <v-flex xs12 sm6>
                                            <v-text-field
                                                    label="Amount Needed (ONE)"
                                                    type="number"
                                                    step="0.0001"
                                                    min="0"
                                                    v-model="newProject.amountGoal">
                                            </v-text-field>
                                        </v-flex>
                                        <v-flex xs12 sm6>
                                            <v-text-field
                                                    label="Duration (in days)"
                                                    type="number"
                                                    v-model="newProject.duration">
                                            </v-text-field>
                                        </v-flex>
                                    </v-layout>
                                </v-container>
                            </v-card-text>
                            <v-card-actions>
                                <v-spacer></v-spacer>
                                <v-btn
                                        color="blue darken-1"
                                        flat
                                        @click="startProjectDialog = false;
                  newProject.isLoading = false;">
                                    Close
                                </v-btn>
                                <v-btn color="blue darken-1"
                                       flat
                                       @click="startProject"
                                       :loading="newProject.isLoading">
                                    Save
                                </v-btn>
                            </v-card-actions>
                        </v-card>
                    </v-dialog>
                </v-layout>
            </v-container>

            <v-container
                    grid-list-lg
            >
                <h1 class="display-1 font-weight-bold mb-3">
                    Projects
                </h1>
                <v-layout row wrap>
                    <v-flex v-for="(project, index) in projectList" :key="index" xs12>
                        <v-dialog
                                v-model="project.dialog"
                                width="800"
                        >
                            <v-card>
                                <v-card-title class="headline font-weight-bold">
                                    {{ project.projectTitle }}
                                </v-card-title>
                                <v-card-text>
                                    {{ project.projectDesc }}
                                </v-card-text>
                                <v-card-actions>
                                    <v-spacer></v-spacer>
                                    <v-btn
                                            color="blue darken-1"
                                            flat="flat"
                                            @click="projectList[index].dialog = false"
                                    >
                                        Close
                                    </v-btn>
                                </v-card-actions>
                            </v-card>
                        </v-dialog>
                        <v-hover>
                            <v-card
                                    slot-scope="{ hover }"
                                    :class="`elevation-${hover ? 10 : 2}`"
                            >
                                <v-card-title primary-title>
                                    <div>
                                        <div class="headline font-weight-bold">
                                            <v-chip
                                                    label
                                                    :color="stateMap[project.currentState].color"
                                                    text-color="white" class="mt-0">
                                                {{ stateMap[project.currentState].text }}
                                            </v-chip>
                                            {{ project.projectTitle }}
                                        </div>
                                        <br/>
                                        <span>{{ project.projectDesc.substring(0, 100) }}</span>
                                        <span v-if="project.projectDesc.length > 100">
                      ... <a @click="projectList[index].dialog = true">[Show full]</a>
                    </span>
                                        <br/><br/>
                                        <small>Up Until: <b>{{ new Date(project.deadline * 1000) }}</b></small>
                                        <br/><br/>
                                        <small>Goal of <b>{{ project.goalAmount / 10**18 }} ONE </b></small>
                                        <small v-if="project.currentState == 1">wasn't achieved before deadline</small>
                                        <small v-if="project.currentState == 2">has been achieved</small>
                                    </div>
                                </v-card-title>
                                <v-flex
                                        v-if="project.currentState == 0 && account != project.projectStarter"
                                        class="d-flex ml-3" xs12 sm6 md3>
                                    <v-text-field
                                            label="Amount (in ONE)"
                                            type="number"
                                            step="0.0001"
                                            min="0"
                                            v-model="project.fundAmount"
                                    ></v-text-field>
                                    <v-btn
                                            class="mt-3"
                                            color="light-blue darken-1 white--text"
                                            @click="fundProject(index)"
                                            :loading="project.isLoading"
                                    >
                                        Fund
                                    </v-btn>
                                </v-flex>
                                <v-flex class="d-flex ml-3" xs12 sm6 md3>
                                    <v-btn
                                            class="mt-3"
                                            color="amber darken-1 white--text"
                                            v-if="project.currentState == 1"
                                            @click="getRefund(index)"
                                            :loading="project.isLoading"
                                    >
                                        Get refund
                                    </v-btn>
                                </v-flex>
                                <v-card-actions v-if="project.currentState == 0" class="text-xs-center">
                  <span class="font-weight-bold" style="width: 200px;">
                    {{ project.currentAmount / 10**18 }} ONE
                  </span>
                                    <v-progress-linear
                                            height="10"
                                            :color="stateMap[project.currentState].color"
                                            :value="(project.currentAmount / project.goalAmount) * 100"
                                    ></v-progress-linear>
                                    <span class="font-weight-bold" style="width: 200px;">
                    {{ project.goalAmount / 10**18 }} ONE
                  </span>
                                </v-card-actions>
                            </v-card>
                        </v-hover>
                    </v-flex>
                </v-layout>
            </v-container>
        </v-content>
    </v-app>
</template>

<script>
    // We import our the scripts for the smart contract instantiation, and web3
    import {getCrowdfundingInstance} from '../contracts/crowdfunding'
    import {getProjectInstance} from "../contracts/project";
    import getExtension from './extension'

    export default {
        name: 'App',
        data() {
            return {
                startProjectDialog: false,
                account: null,
                stateMap: [
                    {color: 'primary', text: 'Ongoing'},
                    {color: 'warning', text: 'Expired'},
                    {color: 'success', text: 'Completed'},
                ],
                projectList: [],
                newProject: {isLoading: false},
                chain: {
                    id: 2,
                    endpoint: "https://api.s0.b.hmny.io",
                    shard: 0,
                },
                transaction: {
                    gasLimit: 6721900,
                    gasPrice: 1000000000
                },
                crowdfundingInstance: null,
                hmyExtension: null,
            };
        },
        mounted() {
            window.addEventListener('load', () => {
                this.hmyExtension = getExtension(this.chain.endpoint, this.chain.shard, this.chain.id)
                this.crowdfundingInstance = getCrowdfundingInstance(this.hmyExtension)
                this.getProjects()
            })
        },
        update() {
            this.getProjects()
        },
        methods: {
            _startProjectWithAccount(account) {
                this.newProject.isLoading = true;
                console.log(this.newProject)
                this.crowdfundingInstance.methods.startProject(
                    this.newProject.title,
                    this.newProject.description,
                    this.newProject.duration,
                    this.hmyExtension.utils.toWei(this.newProject.amountGoal, 'ether')
                ).send({
                    from: account,
                    gasPrice: this.transaction.gasPrice,
                    gasLimit: this.transaction.gasLimit
                }).then((response) => {
                    console.log("Project submitted!")
                    console.log("New Project Contract address: " + response.address)
                    console.log(response.transaction)
                    this.newProject = {isLoading: false};
                    window.location.reload(false);
                }).catch(error => {
                    this.newProject = {isLoading: false};
                    console.error(error)
                    // TODO: splash screen error & message
                    alert(error.message)
                })
            },
            _fundProjectWithAccount(account, index){
                const projectContract = this.projectList[index].contract;
                this.projectList[index].isLoading = true;
                projectContract.methods.contribute().send(
                    {
                        from: account,
                        gasPrice: this.transaction.gasPrice,
                        gasLimit: this.transaction.gasLimit,
                        value: this.hmyExtension.utils.toWei(this.projectList[index].fundAmount, 'ether')
                    }
                ).then(response => {
                    console.log(response.transaction)
                    this.projectList[index] = {isLoading: false};
                    window.location.reload(false);
                }).catch(error => {
                    this.projectList[index] = {isLoading: false};
                    console.error(error)
                    alert(error)
                })
            },
            getProjects() {
                this.crowdfundingInstance.methods.returnAllProjects().call().then((projects) => {
                    projects.forEach((address) => {
                        const projectInstance = getProjectInstance(address,   this.hmyExtension)
                        projectInstance.methods.getDetails().call().then((data) => {
                            const projectData = data;
                            projectData.isLoading = false;
                            projectData.contract = projectInstance
                            this.projectList.push(projectData)
                        })
                    })
                }).catch(error => {
                    console.error(error)
                    // TODO: splash screen error & message
                    alert(error.message)
                })
            },
            startProject() {
                console.log(this.newProject.amount)
                console.log("Adding Project!")
                if (this.account != null) {
                    this._startProjectWithAccount(this.account)
                } else {
                    this.hmyExtension.login().then((acc) => {
                        this.account = acc.account
                        this._startProjectWithAccount(acc.account)
                    })
                }
            },
            fundProject(index) {
                if (!this.projectList[index].fundAmount) {
                    return;
                }
                console.log(this.newProject.amount)
                console.log("Funding Project!")
                if (this.account != null) {
                    this._fundProjectWithAccount(this.account, index)
                } else {
                    this.hmyExtension.login().then((acc) => {
                        this.account = acc.account
                        this._fundProjectWithAccount(acc.account, index)
                    })
                }
            },
            getRefund(index) {
            },
        },
    };
</script>
