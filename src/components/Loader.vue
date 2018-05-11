<template>
	<div>
		<div class="wrapper">
			<div 
				class="column" 
				:class="'column__' + mode[columnKey]" 
				v-for="(column, columnKey) in headers" :key="columnKey"
				>
				<div 
					v-if="mode[columnKey] === 'edit'"
					class="column-top column-top-edit"
					>
					<div class="column-header">
						Column {{newColumnName[columnKey]}}
					</div>
					<select 
						class="form-element column-select" 
						placeholder="Make a selection" 
						v-if="selectColumnName[columnKey] !== 'newColumn'"
						v-model="selectColumnName[columnKey]"
						>
						<option value="" disabled>Create a new column</option>
						<option value="newColumn">New column name</option>
						<option value="" disabled>Available column names</option>
						<option :value="column" v-for="(column, optionKey) in headers" :key="optionKey">
							{{column}}
						</option>
					</select>
					<div v-if="selectColumnName[columnKey] === 'newColumn'">
						<input class="form-element" type="text" v-model="newColumnName[columnKey]">
						<select class="column-select form-element" v-model="columnFieldType[columnKey]">
							<option v-for="(type, fieldKey) in fieldTypes" :key="fieldKey">{{type}}</option>
						</select>
					</div>
					<div class="column-buttons">
						<div class="column-button column-button__button" @click="save(columnKey)">save</div>
						<div v-if="columnKey > 0" class="column-button column-button__button" @click="cancel(columnKey)">back</div>
						<div class="column-button column-button__skip" @click="skip(columnKey)">skip</div>
					</div>
				</div>
				<div 
					v-if="mode[columnKey] === 'saved'"
					class="column-top column-top-saved"
					>
					<div class="column-header">
						Column {{newColumnName[columnKey]}}
						<br>
						<div v-if="columnFieldType[columnKey]">
							field type {{columnFieldType[columnKey]}}
						</div>
					</div>
					<div class="column-buttons">
						<div class="column-button column-button__button" @click="edit(columnKey)">edit</div>
						<div class="column-button column-button__skip" @click="skip(columnKey)">skip</div>
					</div>
				</div>
				<div 
					v-if="mode[columnKey] === 'skiped'"
					class="column-top column-top-saved"
					>
					<div class="column-header">
						Column will not be impotrted
					</div>
					<div class="column-buttons">
						<div class="column-button column-button__button" @click="edit(columnKey)">edit</div>
					</div>
				</div>
				<div
					class="column-top"
					v-if="mode[columnKey] === 'pending'"
					>
					<div class="column-header">
						Unnamed
						<br>
						Unmached collumn
					</div>
					<div class="column-buttons">
						<div class="column-button column-button__button" @click="edit(columnKey)">edit</div>
						<div class="column-button column-button__skip">skip</div>
					</div>
				</div>
				<div class="column-content">
					<div class="row row-header">{{column}}</div>
					<div class="row" v-for="(element, rowKey) in content" :key="rowKey">
						{{element[columnKey]}}
					</div>
				</div>
			</div>
		</div>
		<div v-if="done">
			All columns are matched, click next to finish.
			<button @click="exportData">Next</button>
			<button @click="cancel(headers.length)">Back</button>
		</div>
		<div class="input-file">
			<input type="file" @change="uploadFile($event)">
		</div>
	</div>
</template>

<script>
import csvtojson from'csvtojson'
import readXlsxFile from 'read-excel-file'
export default {
  data () {
    return {
			headers: [],
			content: [],
			selectColumnName: [],
			newColumnName: [],
			editModeOn: 0,
			mode: [],
			columnFieldType: [],
			fieldTypes: ['text', 'data', 'number'],
			savedData: [],
			done: false
    }
	},
	methods: {
		uploadFile(event){
			const file = event.target.files[0]
			const reader = new FileReader()
			const data = []
			const fileTypeArray = file.name.split('').reverse()
			const fileType = fileTypeArray.splice(0, fileTypeArray.indexOf('.')).reverse().join('')
			reader.onload = event => {
				const fileString = event.target.result
				if(fileType === 'xlsx'){
					readXlsxFile(file).then(data => {
						this.headers = data.shift()
						this.content = data
						this.mode[0] = 'edit'
						for(let i=1; i<this.headers.length; i++){
							this.mode[i] = 'pending'
						}
					})
				}
				if(fileType === 'csv'){
					csvtojson({noheader:true})
						.fromString(fileString)
						.on('csv', jsonObject => {
							data.push(jsonObject)
						})
						.on('done', () => {
							this.headers = data.shift()
							this.content = data
							this.mode[0] = 'edit'
							for(let i=1; i<this.headers.length; i++){
								this.mode[i] = 'pending'
							}
						})
				}	
			}
			reader.readAsText(file)	
		},
		setNextUnmatchedToEdit(){
			const unmatched = this.getNextUnmatched()
			if(unmatched){
				this.mode[unmatched] = 'edit'
			}
		},
		save(id){
			let name = ''
			let er = ''
			if(this.selectColumnName[id] === 'newColumn'){
				name = this.newColumnName[id]
			}else{
				name = this.selectColumnName[id]
			}
			if(this.selectColumnName[id] === 'newColumn' && !this.columnFieldType[id]){
				er = 'Set the data type of column'
			}
			if(!name){
				er = 'Set the name of column'
			}
			console.log(er)
			if(!er){
				this.newColumnName[id] = name
				this.savedData[id] = {
					columnName: name,
					data: this.content[id]
				}
				if(this.columnFieldType[id]){
					this.savedData[id].columnType = this.columnFieldType[id]
				}
				this.mode[id] = 'saved'
				this.setNextUnmatchedToEdit()
				this.$forceUpdate()
				this.done = this.mode.every(el => {
					return el === 'saved' || el === 'skiped'
				})
			}else{
				alert(er)
			}
		},
		edit(id){
			for(let column in this.mode){
				if(this.mode[column] === 'edit'){
					this.mode[column] = 'pending'
				}
			}
			this.mode[id] = 'edit'
			this.$forceUpdate()
		},
		getNextUnmatched(){
			let unmatched = []
			this.mode.forEach((el, key) => {
				if(el === 'pending'){
					unmatched.push(key)
				}
			})
			const result = unmatched.reverse().pop()
			if(result){
				return result
			}else{
				return null
			}
		},
		cancel(id){
			if(id > 0){
				this.mode[id] = 'pending'
				this.mode[id-1] = 'edit'
				this.selectColumnName[id-1] = null
				this.selectColumnName[id-1] = null
			}
			this.savedData[id] = {}
			this.columnName = null
			this.selectColumnName[id] = null 
			this.$forceUpdate()
		},
		skip(id){
			this.mode[id] = 'skiped'
			if(id + 1 < this.headers.length){
				this.mode[id+1] = 'edit'
			}
			this.$forceUpdate()
			this.done = this.mode.every(el => {
				return el === 'saved' || el === 'skiped'
			})
		},
		exportData(){
			console.log(this.savedData)
		}
	}
}
</script>
<style lang="sass" scoped>
.input-file
	display: block
.wrapper
	width: auto
	overflow-x: scroll
	white-space: nowrap
	padding: 20px
.row
	padding: 3px
	text-align: left
	font-size: 14px
.column
	width: 240px
	//min-height: 300px
	display: inline-block
	position: relative
	vertical-align: top
	margin-right: 10px
	&__edit
		border: 2px solid #0000ff
		.row
			background-color: lighten(#0000ff, 30)
			border-top: 1px solid #0000ff
		.row-header
			background-color: lighten(#0000ff, 10)
	&__pending
		border: 2px solid #ff0000
		.row
			background-color: lighten(#ff0000, 30)
			border-top: 1px solid #ff000
			border-top: 1px solid #ff0000
		.row-header
			background-color: lighten(#ff0000, 10)
	&__saved
		border: 2px solid #ccc
		.row
			background-color: #eee
			border-top: 1px solid #ccc
		.row-header
			background-color: #eee
	&__skiped
		border: 2px solid #eee
		.row
			border-top: 1px solid #eee
		.row-header
			background-color: #eee
	.column-header
		font-size: 18px
.column-top
	margin-top: 10px
	height: 160px
.column-content
	bottom: 0
.column-buttons
	//width: 200px
	margin:
		top: 20px
		bottom: 20px
	.column-button
		display: inline-block
		cursor: pointer
		text-decoration: underline
		color: #0000ff
	/*.column-button__button
		color: #000
		border: 1px solid #999999
		border-radius: 10px
		padding: 2px 10px
		border: 1px solid #999
		background-color: #eee
		text-decoration: none*/
.form-element
	border: 0
	border: 1px solid #999
	background-color: transparent
	border-radius: 5px
	font-size: 12px
	padding: 2px 15px
	margin: 10px auto
	display: block
.column-select
	margin-bottom: 5px
	background-color: transparent
	border-radius: 5px
	width: 180px
</style>
