console.log('modal:', document.querySelectorAll('#confirmModal').length);
console.log('ok:',    document.querySelectorAll('#confirmOk').length);
document.querySelectorAll('#confirmOk').forEach((el,i)=>
  console.log(i, 'hasOnclick=', !!el.onclick, el));
console.log('bootstrap ver:', window.bootstrap && bootstrap.Modal.VERSION,
            'jqueryModal:', !!(window.jQuery && jQuery.fn.modal));


// 先看有没有上一次的残留
console.log('modal-open=', document.body.classList.contains('modal-open'),
            'backdrops=', document.querySelectorAll('.modal-backdrop').length,
            'shown=', document.querySelectorAll('.modal.show').length);

// 再给 OK 挂一个原生监听，看点击到底有没有到这个节点
var b = document.getElementById('confirmOk');
console.log('onclick已设=', !!b.onclick);
b.addEventListener('click', () => console.log('NATIVE CLICK FIRED'));


return function(confirmMessage, type='info') {
    return new Promise(resolve => {
        $('#confirm_message').text(confirmMessage);
        var $ok = $('#confirmOk');
        $ok.removeClass('btn-primary btn-warning-action btn-inverse-delete').addClass(btnStyle[type]);
        // 关键：先解绑再绑，别用 .onclick=
        $ok.off('click').on('click', function () { $('#confirmModal').modal('hide'); resolve(true); });
        $('#confirmCancel').off('click').on('click', function () { $('#confirmModal').modal('hide'); resolve(false); });
        $('#confirmModal').modal('show');   // 每次从活的 DOM 拿实例，不用缓存
    });
};