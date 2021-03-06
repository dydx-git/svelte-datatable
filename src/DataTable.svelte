<script>
	import { beforeUpdate, createEventDispatcher, onMount } from 'svelte';
	import Fuse from 'fuse.js';

	import Paginate from './Paginate.svelte';
	import { debounce } from './debounce';
	import * as local from './local';
	import { collect, exportExcel, print } from './data-grid';

	import { immerStore } from './stores';

	export let title = '';
	export let customButtons = [];
	export let clickable = true;
	export let printable = true;
	export let exportable = true;
	export let searchable = true;
	export let searching = false;
	export let searchInputRef = null;
	export let searchInput = '';

	export let sortable = true;
	export let sortColumn = -1;
	export let sortIndex = -1;
	export let sortType = 'asc';

	export let currentPerPage = 10;
	export let perPageOptions = [];
	export let currentPage = 1;
	export let rowCount = 0;
	export let pageCount = 0;
	export let selected = 0;
	export let perPage = [10, 20, 30, 40, 50];
	export let paginate = { size: perPage[0], page: 1 };
	export let defaultPerPage = null;
	export let exactSearch = false;		

	export let columns = [];
	export let process = 'server';
	// local
	export let rows = [];
	export let paginatedRows = [];
	// server
	export let paged = { paginate: paginate };
	export let getList = null;

	const dispatch = createEventDispatcher();
	const isServerProcess = process === 'server';

	const paginated = immerStore({
		paginatedRows,
		rows,
		paginate,
		size: currentPerPage,
		page: currentPage,
	});

	export function search(e) {
		searching = !searching;
		if (searching) {
			setTimeout(() => searchInputRef.focus(), 100);
		}
	}

	export function setSortIcon(index) {
		if (!sortable)
			return;
		if (sortColumn === index) {
			sortType = sortType === 'asc' ? 'desc' : 'asc';
		} else {
			sortType = 'asc';
			sortColumn = index;
		}
		sortType = sortType, sortColumn = sortColumn;
	}

	export function setPerPageOptions() {
		let options = perPage;
		// Force numbers
		options = options.map( v => parseInt(v));		
		// Set current page to first value
		currentPerPage = options[0];	
		// Sort options
		options.sort((a,b) => a - b);	
		// And add "All"
		options.push(-1);	
		// If defaultPerPage is provided and it's a valid option, set as current per page
		if (options.indexOf(defaultPerPage) > -1) {
			currentPerPage = parseInt(defaultPerPage);
		}	
		console.log('currentPerPage', currentPerPage, options);
		currentPerPage = currentPerPage, perPageOptions = options;
	}

	export function click(row) {
		if (!clickable) {
			return;
		}
		if (getSelection().toString()) {
			// Return if some text is selected instead of firing the row-click event.
			return;
		}
		dispatch('row-click', row);
		// console.log('click', row);
	}

	export function sort(index) {
		if (index > -1) {
			setSortIcon(index);
			getPaged({colName: columns[index].field, direction: sortType});				
		}
	}

	onMount(() => {
		setPerPageOptions();		
	});

	function getPagedLocal(props, pred) { 		
		const values = local.getPaged(props, pred, {	
			paginate, selected,
			currentPage, currentPerPage,
            sortable, sortColumn, sortType, 
			searchInput, exactSearch,
			rowCount, pageCount, 
			rows, columns	
		});
		({	paginate, selected,
			currentPage, currentPerPage,
            sortable, sortColumn, sortType,
			searchInput, exactSearch,
			rowCount, pageCount,
			rows, paginatedRows
		} = values);
		console.log('values', values);
		update(paginatedRows, currentPerPage, selectedPage);
	}

	function getPagedServer(props, pred, paged, cb) {
        const { paginate, getList } = paged;
        if (getList && (!pred || pred(paginate))) {
            const p = Object.assign({}, paginate, props);
            getList(p).then(data => {
                console.log('getPagedServer ', data );
                cb(data);
            });		
        }
	}
	
	function updateServer(paged) {
		const { paginate } = paged;
		rowCount = paginate.total;
		pageCount = paginate.pages;				
		currentPage = paginate.page;
		if (paged.getList) {
			getList = paged.getList;	
		}
	}

	function updatePaged(data) {
		updateServer(data);
		update(data.rows, data.paginate.size, data.paginate.page);
	}

	function getPaged(props, pred) {
		console.log('props', props);
		if (isServerProcess) {
			getPagedServer(props, pred, { paginate, getList }, updatePaged);
		} else {
			getPagedLocal(props, pred);
		}	
	}	
	
	$: selectedPage = selected + 1;
	let previous;	
	function update(paginatedRows, size, page) {
		paginated.update($paginated => {
			$paginated.paginatedRows = paginatedRows;
			$paginated.rows = rows;
			$paginated.size = size;
			$paginated.page = page;
		});
		previous = $paginated;
	}

	$: {
		// console.log('$paginated - previous - selectedPage ', $paginated.page, previous ? previous.page : -1, selectedPage);
		if (previous) {
			if (!isServerProcess && rows !== previous.rows) {
				getPaged({ rows });
			}
			if (currentPerPage !== previous.size) {
				getPaged({ size: currentPerPage });
			}
			if (selectedPage !== previous.page) {
				getPaged({ page: selectedPage }, x => x.page != selectedPage);
				paginate.page = selectedPage;
			}
		}
		previous = $paginated;
	}

	let prevPaged;
	$: if (paged.rows && paged !== prevPaged) {
		updatePaged(paged);
		prevPaged = paged;
		console.log('paged !== prevPaged ', $paginated );
	}

	let prevSearchInput = '';
	beforeUpdate(() => {			        
		if (searchInput != prevSearchInput) {
			debounce(() => {
				if (searchInput != prevSearchInput) {
					console.log('searchInput changed', searchInput);
					getPaged({ searchText: searchInput });
					prevSearchInput = searchInput;
				}
			}, 400)();
		}	        
	});
</script>

<div class="material-table">
	<div class="table-header">
		<span class="table-title">{title}</span>
		<div class="actions">
			{#each customButtons as button, x}
			{#if !button.hide}
			<a href="#!"
				class="waves-effect btn-flat nopadding"
				on:click="{click}">
				<i class="material-icons">{button.icon}</i>
			</a>
			{/if}
			{/each}
			{#if printable}
			<a href="#!"
				class="waves-effect btn-flat nopadding"
				on:click="{() => print(columns, rows)}">
				<i class="material-icons">print</i>
			</a>
			{/if}
			{#if exportable}
			<a href="#!"
				class="waves-effect btn-flat nopadding"
				v-if="this.exportable"
				on:click="{() => exportExcel(columns, rows, title)}">
				<i class="material-icons">description</i>
			</a>
			{/if}
			{#if searchable}
			<a href="#!"
				class="waves-effect btn-flat nopadding"
				on:click="{search}">
				<i class="material-icons">search</i>
			</a>
			{/if}
		</div>
	</div>
	{#if searching}
	<div>
		<div id="search-input-container">
			<label>
				<input type="search" id="search-input" bind:this={searchInputRef}
				 	class="form-control" placeholder="Search data"
					bind:value="{searchInput}">
			</label>
		</div>
	</div>
	{/if}

    <table  ref="table"
            class="table table-striped table-hover">
        <thead>
            <tr>
                {#each columns as column, x}
                <th on:click="{() => sort(x)}"
                    class="{ sortable ? 'sorting ' : ''}
                        { sortColumn === x ?
                            (sortType === 'desc' ? 'sorting-desc' : 'sorting-asc')
                            : '' }
                        { column.numeric ? ' numeric' : ''}"
                    style="width: { column.width ? column.width : 'auto' }">
                    {column.label}
                </th>
                {/each}
            </tr>
        </thead>

        <tbody>
            {#each $paginated.paginatedRows as row, y}
                <tr class="{ clickable ? 'clickable' : '' }" on:click="{() => click(row)}">
                    {#each columns as column, x}
                        <td>
                            {#if column.html}
                                {@html collect(row, column.field)}
                            {:else}
                                { collect(row, column.field) }   
                            {/if}     
                        </td>    
                    {/each}
                </tr>
            {/each}
        </tbody>
    </table>
{#if paginate}
	<div class="table-footer">
		<div class="datatable-length">
			<label>
				<span>Rows per page:</span>
				<select class="browser-default" bind:value="{currentPerPage}">
					{#each perPageOptions as option, x}
						<option value={option}>
						{ option === -1 ? 'All' : option }
						</option>
					{/each}
				</select>
			</label>
		</div>
		<div class="datatable-info">
			{(currentPage - 1) * currentPerPage ? (currentPage - 1) * currentPerPage : 1}
				-{Math.min(rowCount, currentPerPage * currentPage)} of {rowCount}
		</div>
		<div>
			<Paginate
				bind:pageCount={pageCount}
				marginPages="2"
				pageRange="4"
				containerClass="pagination material-pagination"
				pageLinkClass="waves-effect btn-flat"
				prevLinkClass="waves-effect btn-flat nopadding"
				nextLinkClass="waves-effect btn-flat nopadding"
				bind:selected="{selected}"
				activeClass="active"
				>
				<i class="material-icons" slot="prevContent">chevron_left</i>
				<i class="material-icons" slot="nextContent">chevron_right</i>			
			</Paginate>
		</div>
	</div>
{/if}
</div>

<style>
	div.material-table {
		padding: 0;
	}
	tr.clickable {
		cursor: pointer;
	}
	#search-input {
		margin: 0;
		border: transparent 0 !important;
		height: 48px;
		color: rgba(0, 0, 0, .84);
	}
	#search-input-container {
		padding: 0 14px 0 24px;
		border-bottom: solid 1px #DDDDDD;
	}
	table {
		table-layout: fixed;
	}
	.table-header {
		height: 64px;
		padding-left: 24px;
		padding-right: 14px;
		-webkit-align-items: center;
		-ms-flex-align: center;
		align-items: center;
		display: flex;
		-webkit-display: flex;
		border-bottom: solid 1px #DDDDDD;
	}
	.table-header .actions {
		display: -webkit-flex;
		margin-left: auto;
	}
	.table-header .btn-flat {
		min-width: 36px;
		padding: 0 8px;
	}
	.table-header input {
		margin: 0;
		height: auto;
	}
	.table-header i {
		color: rgba(0, 0, 0, 0.54);
		font-size: 24px;
	}
	.table-footer {
		height: 56px;
		padding-left: 24px;
		padding-right: 14px;
		display: -webkit-flex;
		display: flex;
		-webkit-flex-direction: row;
		flex-direction: row;
		-webkit-justify-content: flex-end;
		justify-content: flex-end;
		-webkit-align-items: center;
		align-items: center;
		font-size: 13px !important;
		color: rgba(0, 0, 0, 0.54);
	}
	.table-footer .datatable-length {
		display: -webkit-flex;
		display: flex;
	}
	.table-footer .datatable-length select {
		outline: none;
	}
	.table-footer label {
		font-size: 13px;
		color: rgba(0, 0, 0, 0.54);
		display: -webkit-flex;
		display: flex;
		-webkit-flex-direction: row;
		/* works with row or column */
		
		flex-direction: row;
		-webkit-align-items: center;
		align-items: center;
		-webkit-justify-content: center;
		justify-content: center;
	}
	.table-footer .select-wrapper {
		display: -webkit-flex;
		display: flex;
		-webkit-flex-direction: row;
		/* works with row or column */
		
		flex-direction: row;
		-webkit-align-items: center;
		align-items: center;
		-webkit-justify-content: center;
		justify-content: center;
	}
	.table-footer .datatable-info,
	.table-footer .datatable-length {
		margin-right: 32px;
	}
	.table-footer .material-pagination {
		display: flex;
		-webkit-display: flex;
		margin: 0;
	}
	.table-footer .material-pagination li {
		color: rgba(0, 0, 0, 0.54);
		padding: 0 2px;
		font-size: 15px;
	}
	.table-footer .material-pagination li a {
		color: rgba(0, 0, 0, 0.54);
		padding: 0 6px;
		font-size: 15px;
	}
	.table-footer .material-pagination li a.nopadding {
		padding: 0;
	}
	.table-footer :global(a) {		
		max-height: 30px;
	}
	.table-footer .select-wrapper input.select-dropdown {
		margin: 0;
		border-bottom: none;
		height: auto;
		line-height: normal;
		font-size: 13px;
		width: 40px;
		text-align: right;
	}
	.table-footer select {
		background-color: transparent;
		width: auto;
		padding: 0;
		border: 0;
		border-radius: 0;
		height: auto;
		margin-left: 20px;
	}
	.table-title {
		font-size: 20px;
		color: #000;
	}
	table tr td {
		padding: 0 0 0 56px;
		height: 48px;
		font-size: 13px;
		color: rgba(0, 0, 0, 0.87);
		border-bottom: solid 1px #DDDDDD;
		white-space: nowrap;
		overflow: hidden;
		text-overflow: ellipsis;
	}
	table td, table th {
    	border-radius: 0;
	}
	table tr td a {
		color: inherit;
	}
	table tr td a i {
		font-size: 18px;
		color: rgba(0, 0, 0, 0.54);
	}
	table tr {
		font-size: 12px;
	}
	table th {
		font-size: 12px;
		font-weight: 500;
		color: #757575;
		cursor: pointer;
		white-space: nowrap;
		padding: 0;
		height: 56px;
		padding-left: 56px;
		vertical-align: middle;
		outline: none !important;
	    overflow: hidden;
	    text-overflow: ellipsis;
	}
	table th:hover {
		overflow: visible;
		text-overflow: initial;
	}
	table th.sorting-asc,
	table th.sorting-desc {
		color: rgba(0, 0, 0, 0.87);
	}
	table th.sorting:after,
	table th.sorting-asc:after  {
		font-family: 'Material Icons';
		font-weight: normal;
		font-style: normal;
		font-size: 16px;
		line-height: 1;
		letter-spacing: normal;
		text-transform: none;
		display: inline-block;
		word-wrap: normal;
		-webkit-font-feature-settings: 'liga';
		-webkit-font-smoothing: antialiased;
		content: "arrow_back";
		-webkit-transform: rotate(90deg);
		display: none;
		vertical-align: middle;
	}
	table th.sorting:hover:after,
	table th.sorting-asc:after,
	table th.sorting-desc:after {
		display: inline-block;
	}
	table th.sorting-desc:after {
		content: "arrow_forward";
	}
	table tbody tr:hover {
		background-color: #EEE;
	}
	
	table th:last-child,
	table td:last-child {
		padding-right: 14px;
	}
	table th:first-child, table td:first-child {
		padding-left: 24px;
	}
</style>