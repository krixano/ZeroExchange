<template>
	<div id="topicQuestion" class="container">
		<div class="row">
	        <div class="col s12 m7 l9">
	        	<nav style="margin-top: 10px; margin-bottom: 15px; background-color: #1976D2;">
	        	    <div class="nav-wrapper">
	        	    	<div class="col s12">
		        	        <a :href="'./?/' + topicAddress" v-on:click.prevent="goto(topicAddress)" class="breadcrumb">{{ topicName.slice(0, topicName.length - 3) }}</a>
		        	        <a :href="'./?/' + topicAddress + '/' + getAuthAddress" v-on:click.prevent="goto(topicAddress + '/' + getAuthAddress)" class="breadcrumb" v-if="question">{{ getName }}</a>
		        	        <a href="#!" class="breadcrumb" v-if="question">{{ question.title }}</a>
	        	    	</div>
	        	    </div>
        		</nav>
	        	<div class="card" v-if="question">
	        		<div class="card-content">
						<div class="chip"><a :href="'./?/' + topicAddress + '/' + getAuthAddress" v-on:click.prevent="goto(topicAddress + '/' + getAuthAddress)">{{ getName }}</a></div>
	        			<span class="card-title" style="margin-left: 10px;">{{ question.title }}</span>
	        			<div v-html="getMarkdown" style="margin-bottom: 5px; margin-left: 10px; font-size: 1.2rem;"></div>
	        			<div style="margin-left: 10px;">Published {{ getDate }} <!--<span>by <a :href="'./?/' + topicAddress + '/' + getAuthAddress" v-on:click.prevent="goto(topicAddress + '/' + getAuthAddress)">{{ getName }}</a></span>--> <em v-if="userIsOwner"> | <a href="#">Edit</a> | <a href="#"> Delete</a></em></div>
	        		</div>
        			<component :is="comment_area" :user-info="userInfo" :current-topic-address="topicAddress" :comments="comments" :reference-id="question.question_id" :reference-auth-address="getAuthAddress" reference-type="q" v-on:update="getComments()">
						<!--<a href="#" style="margin-right: 7px;"><i class="material-icons" style="font-size: 1.3rem;">thumb_up</i></a>
						<a href="#" style="margin-right: 7px;"><i class="material-icons" style="font-size: 1.3rem;">thumb_down</i></a>-->
					</component>
	        	</div>
	        	<h5 v-if="question">Answers <small style="margin-left: 10px; font-size: 65%;"><a :href="'./?/' + topicAddress + '/' + getAuthAddress + '/' + question.question_id + '/answer'" v-on:click.prevent="goto(topicAddress + '/' + getAuthAddress + '/' + question.question_id + '/answer')">Post Answer</a></small></h5>
	        	<div class="divider"></div>

	        	<component :is="answer_list_item" v-for="answer in answers" :key="answer.answer_id" :user-info="userInfo" :merger-zites="mergerZites" :current-topic-address="topicAddress" :show-name="true" :answer="answer" :user-is-question-owner="userIsOwner" v-on:mark-solution="markSolution" v-on:update="getAnswers" :solution-id="question.solution_id" :solution-auth-address="question.solution_auth_address"></component>
	        </div>
	        <div class="col s12 m5 l3">
	        	<component :is="connected_topics" :merger-zites="mergerZites"></component>
	        </div>
	    </div>
	</div>
</template>

<script>
	var moment = require("moment");
	var Router = require("../libs/router.js");
	var connectedTopics = require("../vue_components/connected_topics.vue");
	var AnswerListItem = require("../vue_components/answer_list_item.vue");
	var CommentArea = require("../vue_components/comment_area.vue");

	module.exports = {
		props: ["userInfo", "mergerZites"],
		name: "topicQuestion",
		data: () => {
			return {
				connected_topics: connectedTopics,
				answer_list_item: AnswerListItem,
				comment_area: CommentArea,
				topicName: "",
				topicAddress: "",
				question: null,
				answers: [],
				comments: [],
				showCommentBox: false,
				commentText: "",
				submitBtnDisabled: false
			};
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
			},
			getName: function() {
				if (!this.question || !this.question.cert_user_id) return "";
				var name = this.question.cert_user_id.replace(/@.+/, "");
				return name[0].toUpperCase() + name.slice(1);
			},
			getDate: function() {
				return moment(this.question.date_added).fromNow();
			},
			getAuthAddress: function() {
				return this.question.directory.replace(/data\/users\//, "").replace(/\//g, "");
			},
			getMarkdown: function() {
				return md.render(this.question.body).replace(/(?:(>)|(^|\s|\r\n|\r|\n))(@\S+(?:\'s)?:?)(?:(<)|(\s|$|\r\n|\r|\n))/g, "$1<strong>$2$3$5</strong>$4").replace(/(<(?:p|li|blockquote)>)(\S+(?:\'s)?:)(?:(<)|(\s|$|\r\n|\r|\n))/g, "$1<strong>$2$4</strong>$3");
			},
			userIsOwner: function() {
				if (!this.userInfo) return false;
				return this.getAuthAddress === this.userInfo.auth_address;
			}
		},
		beforeMount: function() {
			var self = this;

			this.$parent.$on("update", function() {
				self.getQuestion();
				self.getComments();
				self.$emit("update");
			});

			this.$parent.$on("setMergerZites", function(mergerZites) {
				self.manageMerger(mergerZites);
				self.getQuestion();
				//self.getAnswers();
				self.getComments();
			});

			// If mergerZites is empty
			if (this.mergerZites && Object.keys(this.mergerZites).length != 0 && this.mergerZites.constructor === Object) {
				this.manageMerger(this.mergerZites);
				self.getQuestion();
				//self.getAnswers();
				self.getComments();
			}
		},
		methods: {
			manageMerger: function(mergerZites) { // TODO: Update this in other pages
				if (this.topicAddress !== "" && this.topicName !== "") return;
				var self = this;
				if (!mergerZites[Router.currentParams["topicaddress"]]) {
					page.addMerger(Router.currentParams["topicaddress"])
						.then((mergerZites) => {
							self.topicName = mergerZites[Router.currentParams["topicaddress"]].content.title + " - ";
							self.topicAddress = Router.currentParams["topicaddress"];
						});
				} else {
					this.topicName = mergerZites[Router.currentParams["topicaddress"]].content.title + " - ";
					this.topicAddress = Router.currentParams["topicaddress"];
				}
			},
			getQuestion: function() {
				var self = this;

				page.getQuestion(this.topicAddress, Router.currentParams["authaddress"], Router.currentParams["questionid"])
					.then((questions) => {
						self.question = questions[0];
						self.getAnswers();
					});
			},
			getAnswers: function() {
				var self = this;
				var solution_id = self.question.solution_id;
				var solution_auth_address = self.question.solution_auth_address;

				page.getQuestionAnswers(this.topicAddress, Router.currentParams["questionid"], Router.currentParams["authaddress"])
					.then((answers) => {
						self.answers = answers.sort(function(a, b) {
							var a_auth_address = a.directory.replace(/data\/users\//, "").replace(/\//g, "");
							var b_auth_address = b.directory.replace(/data\/users\//, "").replace(/\//g, "");
							//console.log(((b.answer_id == solution_id && b_auth_address) == solution_auth_address ? 1 : 0) - ((a.answer_id == solution_id && a_auth_address == solution_auth_address) ? 1 : 0));
							return ((b.answer_id == solution_id && b_auth_address == solution_auth_address) ? 1 : 0) - ((a.answer_id == solution_id && a_auth_address == solution_auth_address) ? 1 : 0);
						});
					});
			},
			getComments: function() {
				var self = this;

				page.getQuestionComments(this.topicAddress, Router.currentParams["questionid"], Router.currentParams["authaddress"])
					.then((comments) => {
						self.comments = comments;
					});
			},
			goto: function(to) {
				Router.navigate(to);
			},
			isActive: function(address) {
				return Router.currentParams["topicaddress"] === address;
			},
			getDate: function(date) {
				return moment(date).fromNow();
			},
			markSolution: function(answer_id, answer_auth_address) {
				var self = this;

				page.questionMarkSolution(this.topicAddress, this.question.question_id, this.getAuthAddress, answer_id, answer_auth_address, function() {
					self.getQuestion();
				});
			}
		}
	}
</script>