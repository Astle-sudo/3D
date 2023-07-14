<script>
// @ts-nocheck

    // @ts-ignore
    /**
     * @type {any}
     */
    export let str;
    export let ID;
    import {Plot} from '../lib/store';
    let exp;
    let userinput = false;

    function onSubmit (e) {
        console.log($Plot)
        const formData = new FormData (e.target);
        userinput = true;
        exp = formData.get("exp");
        for (let plot of $Plot) {
            if (plot.IDnumber == ID) {
                plot.Expression = exp;
            }
        }
    }
    function Remove () {
        userinput = false;
        $Plot = $Plot.filter(function (item){
            return item.IDnumber !== ID;
        });
    } 
</script>

<form class="form" on:submit|preventDefault={onSubmit}>
    <h2 name="str" value = "{str}">
        {str} = 
    </h2>
    <input class="exp" placeholder="expression" name="exp" >
    <input class="Input" type="submit" value="Plot">
    <input class="Input" type="Reset" value="Remove" on:click|preventDefault={Remove}>
</form>

<style>
    h2 {
        margin-top: 0.5%;
    }
    .exp:focus {
        outline: none;
    }
    .form {
        border: 2px solid black;
        margin: 4%;
        padding: 1%;
        width: 26%;
        display: flex;
        flex-direction: row;
        justify-content: space-around;
    }
    .Input {
        padding: 2%;
        color: blue;
        font-size: 0.1em;
        border: 1px solid blue;
        margin: 0 1%;
    }
    .Input:hover {
        background-color: blue;
        color: white;
        cursor: pointer;
    }
</style>