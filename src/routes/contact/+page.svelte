
<svelte:head>
  <title>Contact me</title>
</svelte:head>
<form action="mailto:foo@example.com" method="GET">
    <label for="email">Email: <input id="email" type="email" name="text_email"> </label>
    <label for="subject">Subject: <input id="subject" type="text" name="text_subject"> </label>
    <label for="body">Body: <textarea id="body" name="text_message"></textarea> </label>

    <input type="submit" value="Submit" id="button">
    <!-- <button>Submit</button> -->
</form>
<style>
  form {
    display: grid; /*step 3.3 part 1*/
    grid-template-columns: auto 1fr; /*step 3.3 part 1.1*/
    gap: 1em; /*step 3.3 part 3*/
  }
  form label {
    display: grid;/*step 3.3 part 2*/
    grid-template-columns: subgrid;/*step 3.3 part 2*/
  }
  #button, label{
    grid-column: 1 / -1; /*step 3.3 part 2*/
  }
</style>