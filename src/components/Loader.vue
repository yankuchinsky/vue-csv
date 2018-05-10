<template>
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
					field type {{columnFieldType[columnKey]}}
				</div>
			</div>
			<div 
				v-if="mode[columnKey] === 'skiped'"
				class="column-top column-top-saved"
				>
				<div class="column-header">
					Column {{newColumnName[columnKey]}}
					<br>
					field type {{columnFieldType[columnKey]}}
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
					<div v-if="mode[columnKey] === 'edit'" class="column-button column-button__button">edit</div>
					<div v-if="mode[columnKey] === 'edit'" class="column-button column-button__skip">skip</div>
				</div>
			</div>
			<div class="column-content">
				<div class="row row-header">{{column}}</div>
				<div class="row" v-for="(element, rowKey) in content" :key="rowKey">
					{{element[columnKey]}}
				</div>
			</div>
		</div>
		<input type="file" @change="uploadFile($event)">
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
			savedData: []
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
				if(file.type === 'csv'){
					csvtojson({noheader:true})
						.fromString(fileString)
						.on('csv', jsonObject => {
							data.push(jsonObject)
						})
						.on('done', () => {
							//console.log(data)
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
		save(id){
			let name = ''
			if(this.selectColumnName[id] === 'newColumn'){
				name = this.newColumnName[id]
			}else{
				name = this.selectColumnName[id]
			}
			this.newColumnName[id] = name
			this.savedData[id] = {
				columnName: name,
				columnType: this.columnFieldType[id],
				data: this.content[id]
			}
			this.mode[id] = 'saved'
			this.mode[id+1] = 'edit'
			this.$forceUpdate()
			console.log(this.savedData[id])
		},
		cancel(id){
			if(id > 0){
				this.mode[id] = 'pending'
				this.mode[id-1] = 'edit'
			}
			this.savedData[id] = {}
			this.columnName = null
			this.selectColumnName[id] = null 
			this.$forceUpdate()
		},
		skip(id){
			this.mode[id] = 'skiped'
			this.mode[id+1] = 'edit'
			this.$forceUpdate()
		}
	}
}
</script>
<style lang="sass" scoped>
.wrapper
	width: auto
	overflow-x: scroll
	white-space: nowrap
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
			background-color: #eee //#eee
	.column-header
		font-size: 18px	
		//border-bottom: 1px solid #eeeeee
.column-top
	margin-top: 10px
	height: 160px
.column-content
	//position: absolute
	//width: 200px
	bottom: 0
.column-buttons
	width: 200px
	margin:
		top: 20px
		bottom: 20px
	.column-button
		display: inline-block
		cursor: pointer
		text-decoration: underline
		color: #0000ff
	.column-button__button
		color: #000
		border: 1px solid #999999
		border-radius: 10px
		padding: 2px 10px
		border: 1px solid #999
		background-color: #eee
		text-decoration: none
.form-element
	border: 0
	border: 1px solid #999
	margin-bottom: 5px
	background-color: transparent
	border-radius: 5px
	font-size: 12px
	padding: 2px 15px
.column-select
	margin-bottom: 5px
	background-color: transparent
	border-radius: 5px
	width: 180px
</style>
