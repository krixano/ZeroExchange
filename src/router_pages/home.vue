<template>
	<div id="mainapp" class="container">
		<div class="row">
	        <div class="col s12 m7 l9">
	        	<component :is="home_navbar" active="top" :user-info="userInfo"></component>

				<h5>Recent Questions</h5>
	        	<div class="divider"></div>

				<component :is="question_list_item" v-for="question in recentQuestions" :key="question.question_id" :user-info="userInfo" :merger-zites="mergerZites" :question="question" :show-name="true" :current-topic-address="topicAddress" :show-topic-name="true" v-on:update="getQuestions"></component>
	        </div>
	        <div class="col s12 m5 l3">
	        	<component :is="connected_topics" :merger-zites="mergerZites"></component>
	        </div>
	    </div>
	</div>
</template>

<script>
	var Router = require("../libs/router.js");
	var connectedTopics = require("../vue_components/connected_topics.vue");
	var HomeNavbar = require("../vue_components/home_navbar.vue");
	var QuestionListItem = require("../vue_components/question_list_item.vue");

	module.exports = {
		props: ["userInfo", "mergerZites"],
		name: "mainapp",
		data: () => {
			return {
				connected_topics: connectedTopics,
				home_navbar: HomeNavbar,
				question_list_item: QuestionListItem,
				recentQuestions: []
			}
		},
		beforeMount: function() {
			var self = this;

			this.$parent.$on("update", function() {
				self.getQuestions();
			});

			this.$parent.$on("setMergerZites", function(mergerZites) {
				self.manageMerger(mergerZites);
				self.getQuestions();
			});

			// If mergerZites is not empty
			if (this.mergerZites && Object.keys(this.mergerZites).length != 0 && this.mergerZites.constructor === Object) {
				this.manageMerger(this.mergerZites);
				this.getQuestions();
			}
		},
		computed: {
			mergerZiteValues: function() {
				if (this.mergerZites == null) {
					return [];
				}
				return Object.values(this.mergerZites);
			},
			mergerZiteKeys: function() {
				if (this.mergerZites == null) {
					return [];
				}
				return Object.keys(this.mergerZites);
			}
		},
		methods: {
			manageMerger: function(mergerZites) {
				//mergerZites = { "17PRT7jHB4TN1PMzgWbxDQYrUnWKX2bNcM": "ZeroExchange" };
				if (Object.keys(mergerZites).length == 1 && mergerZites["17PRT7jHB4TN1PMzgWbxDQYrUnWKX2bNcM"]) {
					Router.navigate('top-available');
				} else if (Object.keys(mergerZites).length <= 0) {
					Router.navigate('top-available');
				}
			},
			goto: function(to) {
				Router.navigate(to);
			},
			getQuestions: function() {
                var self = this;

                page.getQuestionsRecent()
                    .then((questions) => {
                        self.recentQuestions = questions;
                    });
            }
		}
	}
</script>