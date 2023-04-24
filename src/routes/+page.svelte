<script lang="ts">
	import ChatMessage from '$lib/components/ChatMessage.svelte'
	import type { ChatCompletionRequestMessage } from 'openai'
	import { SSE } from 'sse.js'
  
	let query: string = ''
	let answer: string = ''
	let loading: boolean = false
	let chatMessages: ChatCompletionRequestMessage[] = JSON.parse(localStorage.getItem('chatMessages') || '[]')
	let scrollToDiv: HTMLDivElement
  
	function scrollToBottom() {
	  setTimeout(function () {
		scrollToDiv.scrollIntoView({ behavior: 'smooth', block: 'end', inline: 'nearest' })
	  }, 100)
	}
  
	const handleSubmit = async () => {
	  loading = true
	  chatMessages = [...chatMessages, { role: 'user', content: query }]
  
	  const eventSource = new SSE('/api/chat', {
		headers: {
		  'Content-Type': 'application/json'
		},
		payload: JSON.stringify({ messages: chatMessages })
	  })
  
	  query = ''
  
	  eventSource.addEventListener('error', handleError)
  
	  eventSource.addEventListener('message', (e) => {
		scrollToBottom()
		try {
		  loading = false
		  if (e.data === '[DONE]') {
			chatMessages = [...chatMessages, { role: 'assistant', content: answer }]
			answer = ''
			localStorage.setItem('chatMessages', JSON.stringify(chatMessages))
			return
		  }
  
		  const completionResponse = JSON.parse(e.data)
		  const [{ delta }] = completionResponse.choices
  
		  if (delta.content) {
			answer = (answer ?? '') + delta.content
		  }
		} catch (err) {
		  handleError(err)
		}
	  })
	  eventSource.stream()
	  scrollToBottom()
	}
  
	function handleError<T>(err: T) {
	  loading = false
	  query = ''
	  answer = ''
	  console.error(err)
	}
  </script>
  
  <div class="flex flex-col pt-5 w-full px-0 items-center gap-2">
	<div class="h-[100%] w-full bg-gray-900 rounded-md p-4 overflow-y-auto flex flex-col gap-4">
	  <div class="flex flex-col gap-2">
		{#if chatMessages.length === 0}
		  <ChatMessage type="assistant" message="Hello! How can I assist you today?" />
		{/if}
		{#each chatMessages as message}
		  <ChatMessage type={message.role} message={message.content} />
		{/each}
		{#if answer}
		  <ChatMessage type="assistant" message={answer} />
		{/if}
		{#if loading}
		  <ChatMessage type="assistant" message="Loading.." />
		{/if}
	  </div>
	  <div class="" bind:this={scrollToDiv} />
	</div>
	<form class="flex w-full rounded-md gap-4 bg-gray-900 p-4" on:submit|preventDefault={() => handleSubmit()}>
	  <input type="text" class="input input-bordered w-full" bind:value={query} />
	  <button type="submit" class="btn btn-accent"> Send </button>
	</form>
  </div>
  